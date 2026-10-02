# Part 024 — MongoDB + Mongoose Integration 🍃

> **ระดับ:** Intermediate-Advanced | **เวลาเรียน:** 120 นาที | **ขั้นตอนที่:** 831–870

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Mongoose setup
- Schema definition
- CRUD operations
- Aggregation pipeline
- Text search
- Indexes
- Populate (joins)
- Lean queries
- Transaction ด้วย sessions

---

## 📌 Step 831: Mongoose Setup

```bash
npm install mongoose
```

```javascript
// src/db/mongodb.js
import mongoose from 'mongoose';

export async function connectDB() {
  await mongoose.connect(process.env.MONGODB_URI, {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
  });
  
  mongoose.connection.on('error', (err) => {
    console.error('MongoDB error:', err);
  });
  
  console.log('MongoDB connected');
}
```

---

## 📌 Step 832: Mongoose Models

```javascript
// src/models/user.model.js
import mongoose from 'mongoose';
import bcrypt from 'bcryptjs';

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true,
    minlength: 2,
    maxlength: 100,
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    index: true,
  },
  password: {
    type: String,
    required: true,
    minlength: 8,
    select: false, // ไม่ return ใน queries
  },
  role: {
    type: String,
    enum: ['ADMIN', 'USER'],
    default: 'USER',
    index: true,
  },
  avatar: String,
  bio: String,
  isActive: { type: Boolean, default: true, index: true },
}, {
  timestamps: true,  // createdAt, updatedAt
  toJSON: {
    transform(_, ret) {
      ret.id = ret._id;
      delete ret._id;
      delete ret.__v;
      delete ret.password;
      return ret;
    },
  },
});

// Indexes
userSchema.index({ name: 'text', bio: 'text' });

// Hash password ก่อน save
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

userSchema.methods.comparePassword = function(password) {
  return bcrypt.compare(password, this.password);
};

userSchema.methods.toGraphQL = function() {
  return {
    id: this._id.toString(),
    name: this.name,
    email: this.email,
    role: this.role,
    avatar: this.avatar,
    bio: this.bio,
    isActive: this.isActive,
    createdAt: this.createdAt.toISOString(),
    updatedAt: this.updatedAt.toISOString(),
  };
};

export const User = mongoose.model('User', userSchema);
```

```javascript
// src/models/post.model.js
const postSchema = new mongoose.Schema({
  title: { type: String, required: true, trim: true },
  slug: { type: String, required: true, unique: true, index: true },
  content: { type: String, required: true },
  excerpt: String,
  status: {
    type: String,
    enum: ['DRAFT', 'PUBLISHED', 'ARCHIVED'],
    default: 'DRAFT',
    index: true,
  },
  publishedAt: { type: Date, index: true },
  
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
    index: true,
  },
  
  tags: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Tag' }],
  
  viewCount: { type: Number, default: 0 },
  likeCount: { type: Number, default: 0 },
  
  // Embedded subdocument (ไม่ใช่ reference)
  featuredImage: {
    url: String,
    alt: String,
    width: Number,
    height: Number,
  },
}, {
  timestamps: true,
  toJSON: {
    transform(_, ret) {
      ret.id = ret._id;
      delete ret._id;
      delete ret.__v;
      return ret;
    },
  },
});

// Full-text search index
postSchema.index({ title: 'text', content: 'text', excerpt: 'text' });

// Compound index
postSchema.index({ status: 1, publishedAt: -1 });

export const Post = mongoose.model('Post', postSchema);
```

---

## 📌 Step 833: CRUD Resolvers

```javascript
// src/resolvers/post.resolver.js
export const postResolvers = {
  Query: {
    post: async (_, { id }) => {
      return Post.findById(id)
        .populate('author', 'id name email avatar')
        .populate('tags')
        .lean(); // ← เร็วกว่า Mongoose documents
    },
    
    posts: async (_, { limit = 10, offset = 0, status = 'PUBLISHED' }) => {
      const filter = { status };
      
      const [posts, total] = await Promise.all([
        Post.find(filter)
          .sort({ publishedAt: -1 })
          .skip(offset)
          .limit(limit)
          .populate('author', 'id name avatar')
          .populate('tags')
          .lean(),
        Post.countDocuments(filter),
      ]);
      
      return {
        posts: posts.map(p => ({ ...p, id: p._id.toString() })),
        total,
        hasMore: offset + limit < total,
      };
    },
    
    searchPosts: async (_, { term, limit = 10 }) => {
      return Post.find(
        { $text: { $search: term }, status: 'PUBLISHED' },
        { score: { $meta: 'textScore' } }
      )
        .sort({ score: { $meta: 'textScore' } })
        .limit(limit)
        .populate('author')
        .lean();
    },
  },
  
  Mutation: {
    createPost: async (_, { input }, { user }) => {
      if (!user) throw new GraphQLError('Unauthorized');
      
      const slug = generateSlug(input.title) + '-' + Date.now();
      
      const post = await Post.create({
        ...input,
        slug,
        author: user.sub,
        status: 'DRAFT',
      });
      
      return post.populate('author').populate('tags');
    },
    
    updatePost: async (_, { id, input }, { user }) => {
      const post = await Post.findById(id);
      if (!post) throw new GraphQLError('Post not found');
      if (post.author.toString() !== user.sub && user.role !== 'ADMIN') {
        throw new GraphQLError('Forbidden');
      }
      
      Object.assign(post, input);
      await post.save();
      
      return post.populate('author').populate('tags');
    },
  },
};
```

---

## 📌 Step 834: Aggregation Pipeline

```javascript
const resolvers = {
  Query: {
    // Dashboard stats
    stats: async (_, __, { user }) => {
      const [userStats, postStats, popularTags] = await Promise.all([
        // User stats
        User.aggregate([
          {
            $group: {
              _id: '$role',
              count: { $sum: 1 },
            },
          },
        ]),
        
        // Post stats
        Post.aggregate([
          {
            $group: {
              _id: '$status',
              count: { $sum: 1 },
              totalViews: { $sum: '$viewCount' },
              totalLikes: { $sum: '$likeCount' },
            },
          },
        ]),
        
        // Popular tags
        Post.aggregate([
          { $match: { status: 'PUBLISHED' } },
          { $unwind: '$tags' },
          { $group: { _id: '$tags', count: { $sum: 1 } } },
          { $sort: { count: -1 } },
          { $limit: 10 },
          { $lookup: { from: 'tags', localField: '_id', foreignField: '_id', as: 'tag' } },
          { $unwind: '$tag' },
          { $project: { tag: 1, count: 1 } },
        ]),
      ]);
      
      return { userStats, postStats, popularTags };
    },
    
    // Monthly post counts
    postsByMonth: async (_, { year }) => {
      return Post.aggregate([
        {
          $match: {
            status: 'PUBLISHED',
            publishedAt: {
              $gte: new Date(`${year}-01-01`),
              $lt: new Date(`${year + 1}-01-01`),
            },
          },
        },
        {
          $group: {
            _id: { $month: '$publishedAt' },
            count: { $sum: 1 },
            views: { $sum: '$viewCount' },
          },
        },
        { $sort: { '_id': 1 } },
        { $project: { month: '$_id', count: 1, views: 1, _id: 0 } },
      ]);
    },
  },
};
```

---

## 📌 Step 835: Transactions (MongoDB Sessions)

```javascript
const resolvers = {
  Mutation: {
    transferCredits: async (_, { fromUserId, toUserId, amount }) => {
      const session = await mongoose.startSession();
      
      try {
        await session.withTransaction(async () => {
          // ลด credits ผู้ส่ง
          const fromUser = await User.findByIdAndUpdate(
            fromUserId,
            { $inc: { credits: -amount } },
            { session, new: true }
          );
          
          if (fromUser.credits < 0) {
            throw new Error('Insufficient credits');
          }
          
          // เพิ่ม credits ผู้รับ
          await User.findByIdAndUpdate(
            toUserId,
            { $inc: { credits: amount } },
            { session }
          );
          
          // Log transaction
          await Transaction.create([{
            fromUserId,
            toUserId,
            amount,
            type: 'TRANSFER',
          }], { session });
        });
        
        return { success: true };
      } finally {
        await session.endSession();
      }
    },
  },
};
```

---

## 📌 Step 836: สรุป Part 024

### เนื้อหาที่เรียนรู้

✅ Mongoose setup + models  
✅ Schema definition + indexes  
✅ CRUD operations  
✅ .lean() สำหรับ performance  
✅ populate สำหรับ joins  
✅ Aggregation pipeline  
✅ Full-text search  
✅ Transactions ด้วย sessions  

### Homework

1. สร้าง blog API ด้วย MongoDB + Mongoose
2. implement aggregation สำหรับ dashboard
3. ใช้ transactions สำหรับ order creation

### ในส่วนถัดไป

➡️ **[Part 025](./part-025.md)** — Redis Integration & Real-time Features

---

*Part 024 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
