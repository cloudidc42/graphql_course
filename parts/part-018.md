# Part 018 — File Upload สำหรับ GraphQL 📁

> **ระดับ:** Intermediate | **เวลาเรียน:** 90 นาที | **ขั้นตอนที่:** 591–630

---

## 🎯 สิ่งที่จะได้เรียนรู้

- GraphQL Multipart Request Spec
- graphql-upload-minimal
- Upload scalar
- Single และ Multiple file uploads
- File validation
- S3/CloudFlare R2 integration
- Image processing (Sharp)
- Progress tracking

---

## 📌 Step 591: Upload Schema

```graphql
scalar Upload

type FileUploadResult {
  id: ID!
  url: String!
  filename: String!
  mimetype: String!
  size: Int!
}

type Mutation {
  uploadAvatar(file: Upload!): FileUploadResult!
  uploadProductImages(files: [Upload!]!): [FileUploadResult!]!
  createPostWithCover(input: CreatePostInput!, cover: Upload): Post!
}
```

---

## 📌 Step 592: Server Setup

```bash
npm install graphql-upload-minimal
```

```javascript
// src/server.js
import express from 'express';
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { graphqlUploadExpress } from 'graphql-upload-minimal';

const app = express();

// ต้องใส่ก่อน Apollo middleware
app.use(
  graphqlUploadExpress({
    maxFileSize: 10 * 1024 * 1024,  // 10 MB
    maxFiles: 10,                    // สูงสุด 10 files
  })
);

app.use('/graphql', expressMiddleware(server));
```

---

## 📌 Step 593: Upload Resolver

```javascript
// src/resolvers/upload.resolver.js
import path from 'path';
import { createWriteStream, mkdir } from 'fs';
import { promisify } from 'util';
import { v4 as uuidv4 } from 'uuid';
import sharp from 'sharp';

const mkdirAsync = promisify(mkdir);

// Allowed MIME types
const ALLOWED_IMAGE_TYPES = [
  'image/jpeg',
  'image/jpg',
  'image/png',
  'image/webp',
  'image/gif',
];

const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10 MB

async function processAndSaveImage(fileStream, mimetype, uploadDir) {
  const fileId = uuidv4();
  const ext = mimetype.split('/')[1];
  const filename = `${fileId}.webp`; // แปลงเป็น webp เสมอ
  
  await mkdirAsync(uploadDir, { recursive: true });
  const outputPath = path.join(uploadDir, filename);
  
  // ใช้ Sharp สำหรับ image processing
  await new Promise((resolve, reject) => {
    fileStream
      .pipe(
        sharp()
          .resize(1920, 1080, {
            fit: 'inside',
            withoutEnlargement: true,
          })
          .webp({ quality: 85 })
      )
      .pipe(createWriteStream(outputPath))
      .on('finish', resolve)
      .on('error', reject);
  });
  
  const stats = await fs.promises.stat(outputPath);
  
  return {
    id: fileId,
    filename,
    url: `/uploads/${filename}`,
    mimetype: 'image/webp',
    size: stats.size,
  };
}

export const uploadResolvers = {
  Mutation: {
    uploadAvatar: async (_, { file }, { user }) => {
      if (!user) throw new GraphQLError('Unauthorized');
      
      // file = Promise ที่ resolve เป็น { createReadStream, filename, mimetype, encoding }
      const { createReadStream, filename, mimetype } = await file;
      
      // Validate MIME type
      if (!ALLOWED_IMAGE_TYPES.includes(mimetype)) {
        throw new GraphQLError(
          `File type ${mimetype} not allowed. Allowed types: ${ALLOWED_IMAGE_TYPES.join(', ')}`,
          { extensions: { code: 'BAD_USER_INPUT' } }
        );
      }
      
      const stream = createReadStream();
      const uploadDir = path.join(process.cwd(), 'uploads', 'avatars');
      
      const result = await processAndSaveImage(stream, mimetype, uploadDir);
      
      // อัพเดท user avatar ใน DB
      await User.update(
        { avatar: result.url },
        { where: { id: user.sub } }
      );
      
      return result;
    },
    
    uploadProductImages: async (_, { files }, { user }) => {
      if (!user) throw new GraphQLError('Unauthorized');
      if (files.length > 10) throw new GraphQLError('Maximum 10 files');
      
      const uploadDir = path.join(process.cwd(), 'uploads', 'products');
      
      const results = await Promise.all(
        files.map(async (filePromise) => {
          const { createReadStream, filename, mimetype } = await filePromise;
          
          if (!ALLOWED_IMAGE_TYPES.includes(mimetype)) {
            throw new GraphQLError(`Invalid file type: ${filename}`);
          }
          
          const stream = createReadStream();
          return processAndSaveImage(stream, mimetype, uploadDir);
        })
      );
      
      return results;
    },
  },
};
```

---

## 📌 Step 594: S3 Upload

```javascript
// src/services/s3.service.js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { v4 as uuidv4 } from 'uuid';

const s3 = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
});

export async function uploadToS3(stream, mimetype, folder = 'uploads') {
  const fileId = uuidv4();
  const ext = mimetype.split('/')[1];
  const key = `${folder}/${fileId}.${ext}`;
  
  // อ่าน stream ทั้งหมดก่อน upload
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  const buffer = Buffer.concat(chunks);
  
  await s3.send(new PutObjectCommand({
    Bucket: process.env.AWS_BUCKET_NAME,
    Key: key,
    Body: buffer,
    ContentType: mimetype,
    ACL: 'public-read',
  }));
  
  const url = `https://${process.env.AWS_BUCKET_NAME}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
  
  return {
    id: fileId,
    url,
    filename: `${fileId}.${ext}`,
    mimetype,
    size: buffer.length,
  };
}
```

---

## 📌 Step 595: Client-side Upload

```javascript
// React upload component
import { useMutation, gql } from '@apollo/client';

const UPLOAD_AVATAR = gql`
  mutation UploadAvatar($file: Upload!) {
    uploadAvatar(file: $file) {
      id
      url
      filename
    }
  }
`;

export function AvatarUpload({ onSuccess }) {
  const [uploadAvatar, { loading }] = useMutation(UPLOAD_AVATAR, {
    onCompleted: ({ uploadAvatar }) => {
      onSuccess(uploadAvatar.url);
    },
  });
  
  const [preview, setPreview] = useState(null);
  const [progress, setProgress] = useState(0);
  
  async function handleChange(e) {
    const file = e.target.files[0];
    if (!file) return;
    
    // Preview
    const reader = new FileReader();
    reader.onload = () => setPreview(reader.result);
    reader.readAsDataURL(file);
    
    // Upload
    await uploadAvatar({
      variables: { file },
      context: {
        fetchOptions: {
          onUploadProgress: (progress) => {
            setProgress(Math.round((progress.loaded / progress.total) * 100));
          },
        },
      },
    });
  }
  
  return (
    <div>
      {preview && (
        <img src={preview} alt="Preview" className="w-32 h-32 rounded-full" />
      )}
      
      {loading && (
        <div className="progress-bar">
          <div style={{ width: `${progress}%` }} />
          <span>{progress}%</span>
        </div>
      )}
      
      <input
        type="file"
        accept="image/*"
        onChange={handleChange}
        disabled={loading}
      />
    </div>
  );
}
```

---

## 📌 Step 596: สรุป Part 018

### เนื้อหาที่เรียนรู้

✅ GraphQL Multipart Request Spec  
✅ Upload scalar + server setup  
✅ File validation (type, size)  
✅ Image processing ด้วย Sharp  
✅ S3 integration  
✅ Client-side upload ด้วย Apollo Client  
✅ Progress tracking  

### Homework

1. implement avatar upload ด้วย S3
2. เพิ่ม image resizing (thumbnail generation)
3. เพิ่ม file size และ type validation

### ในส่วนถัดไป

➡️ **[Part 019](./part-019.md)** — Caching Strategies

---

*Part 018 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
