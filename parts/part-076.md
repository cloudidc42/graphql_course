# Part 076 — GraphQL File Upload & Binary Data 📁

> **ระดับ:** Expert | **เวลาเรียน:** 75 นาที | **ขั้นตอนที่:** 1941–1975

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Multipart upload specification
- S3 presigned URL pattern
- Direct-to-CDN upload
- Image processing pipeline
- Video upload & transcoding
- File validation & security
- Progress tracking
- Batch uploads

---

## 📌 Step 1941: Upload Strategies

```
Strategy 1: GraphQL Multipart (graphql-upload)
- ส่งไฟล์ผ่าน GraphQL mutation โดยตรง
- ✅ ง่ายสำหรับ small files
- ❌ ผ่าน server → ช้า, memory intensive
- ❌ Apollo Client ไม่รองรับ natively

Strategy 2: Presigned URL (recommended)
- Server สร้าง presigned URL
- Client upload ตรงไปยัง S3/GCS
- Server เก็บ URL หลัง upload
- ✅ ไม่ผ่าน server → เร็ว, ประหยัด bandwidth
- ✅ Client ส่งได้ถึง 5GB
- ✅ Progress tracking ง่าย

Strategy 3: CDN Direct Upload (Cloudflare Images, etc.)
- Upload ตรงไป CDN
- Auto-resize, optimize
```

---

## 📌 Step 1942: Presigned URL Pattern

```graphql
# schema.graphql

type Mutation {
  # Step 1: Request presigned URL
  requestUploadUrl(input: RequestUploadUrlInput!): UploadUrlResult!
  
  # Step 2: Client uploads ตรงไป S3 (ไม่ผ่าน GraphQL)
  
  # Step 3: Confirm upload
  confirmUpload(uploadId: String!, metadata: UploadMetadataInput): UploadedFile!
}

input RequestUploadUrlInput {
  filename: String!
  contentType: String!
  fileSize: Int!  # bytes
  purpose: UploadPurpose!
}

enum UploadPurpose {
  PRODUCT_IMAGE
  USER_AVATAR
  DOCUMENT
  VIDEO
}

type UploadUrlResult {
  uploadId: String!
  url: String!          # S3 presigned URL
  fields: JSONObject    # Form fields สำหรับ multipart upload
  expiresAt: DateTime!
}

type UploadedFile {
  id: ID!
  url: String!
  thumbnailUrl: String
  filename: String!
  contentType: String!
  fileSize: Int!
  uploadedAt: DateTime!
}

scalar JSONObject
```

```typescript
// src/resolvers/upload.resolver.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { createPresignedPost } from '@aws-sdk/s3-presigned-post';

const s3 = new S3Client({ region: process.env.AWS_REGION });

const ALLOWED_TYPES: Record<string, string[]> = {
  PRODUCT_IMAGE: ['image/jpeg', 'image/png', 'image/webp'],
  USER_AVATAR: ['image/jpeg', 'image/png', 'image/gif'],
  DOCUMENT: ['application/pdf', 'application/msword'],
  VIDEO: ['video/mp4', 'video/quicktime'],
};

const MAX_SIZES: Record<string, number> = {
  PRODUCT_IMAGE: 10 * 1024 * 1024,    // 10MB
  USER_AVATAR: 2 * 1024 * 1024,        // 2MB
  DOCUMENT: 50 * 1024 * 1024,          // 50MB
  VIDEO: 2 * 1024 * 1024 * 1024,      // 2GB
};

export const uploadResolvers = {
  Mutation: {
    requestUploadUrl: async (_, { input }, ctx) => {
      requireAuth(ctx);
      
      const { filename, contentType, fileSize, purpose } = input;
      
      // Validate
      const allowedTypes = ALLOWED_TYPES[purpose] ?? [];
      if (!allowedTypes.includes(contentType)) {
        throw new AppError(
          `Content type ${contentType} not allowed for ${purpose}`,
          ErrorCode.VALIDATION_ERROR
        );
      }
      
      const maxSize = MAX_SIZES[purpose] ?? 0;
      if (fileSize > maxSize) {
        throw new AppError(
          `File size ${fileSize} exceeds limit ${maxSize}`,
          ErrorCode.VALIDATION_ERROR
        );
      }
      
      const uploadId = crypto.randomUUID();
      const ext = filename.split('.').pop() ?? '';
      const key = `uploads/${ctx.user!.id}/${uploadId}.${ext}`;
      
      // Create presigned POST (สำหรับ browser upload)
      const { url, fields } = await createPresignedPost(s3, {
        Bucket: process.env.S3_BUCKET!,
        Key: key,
        Conditions: [
          { 'Content-Type': contentType },
          ['content-length-range', 1, fileSize],
          { 'x-amz-meta-user-id': ctx.user!.id },
          { 'x-amz-meta-upload-id': uploadId },
        ],
        Fields: {
          'Content-Type': contentType,
          'x-amz-meta-user-id': ctx.user!.id,
          'x-amz-meta-upload-id': uploadId,
        },
        Expires: 900,  // 15 minutes
      });
      
      // Store pending upload ใน DB
      await ctx.prisma.pendingUpload.create({
        data: {
          id: uploadId,
          userId: ctx.user!.id,
          s3Key: key,
          filename,
          contentType,
          fileSize,
          purpose,
          expiresAt: new Date(Date.now() + 900_000),
        },
      });
      
      return {
        uploadId,
        url,
        fields,
        expiresAt: new Date(Date.now() + 900_000),
      };
    },
    
    confirmUpload: async (_, { uploadId, metadata }, ctx) => {
      requireAuth(ctx);
      
      const pending = await ctx.prisma.pendingUpload.findFirst({
        where: {
          id: uploadId,
          userId: ctx.user!.id,
          confirmedAt: null,
          expiresAt: { gt: new Date() },
        },
      });
      
      if (!pending) {
        throw Errors.notFound('Upload', uploadId);
      }
      
      // Verify object exists in S3
      try {
        await s3.send(new HeadObjectCommand({
          Bucket: process.env.S3_BUCKET!,
          Key: pending.s3Key,
        }));
      } catch {
        throw new AppError('Upload not found in storage', ErrorCode.PRECONDITION_FAILED);
      }
      
      // Create permanent file record
      const cdnUrl = `${process.env.CDN_URL}/${pending.s3Key}`;
      
      const file = await ctx.prisma.$transaction(async (tx) => {
        // Confirm upload
        await tx.pendingUpload.update({
          where: { id: uploadId },
          data: { confirmedAt: new Date() },
        });
        
        // Create file record
        return tx.uploadedFile.create({
          data: {
            userId: ctx.user!.id,
            s3Key: pending.s3Key,
            url: cdnUrl,
            filename: pending.filename,
            contentType: pending.contentType,
            fileSize: pending.fileSize,
            purpose: pending.purpose,
            altText: metadata?.altText,
          },
        });
      });
      
      // Trigger async processing
      if (pending.purpose === 'PRODUCT_IMAGE') {
        await imageProcessingQueue.add('process-image', {
          fileId: file.id,
          s3Key: pending.s3Key,
        });
      }
      
      return file;
    },
  },
};
```

---

## 📌 Step 1943: Image Processing Pipeline

```typescript
// src/workers/image-processor.ts
import Sharp from 'sharp';
import { S3Client, GetObjectCommand, PutObjectCommand } from '@aws-sdk/client-s3';

interface ImageVariant {
  name: string;
  width: number;
  height?: number;
  format: 'webp' | 'jpeg' | 'avif';
  quality: number;
}

const VARIANTS: ImageVariant[] = [
  { name: 'thumbnail', width: 150, height: 150, format: 'webp', quality: 80 },
  { name: 'medium', width: 600, format: 'webp', quality: 85 },
  { name: 'large', width: 1200, format: 'webp', quality: 90 },
  { name: 'original', width: 2400, format: 'webp', quality: 92 },
];

export async function processProductImage(fileId: string, s3Key: string) {
  // Download original
  const getResponse = await s3.send(new GetObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: s3Key,
  }));
  
  const originalBuffer = await streamToBuffer(getResponse.Body as NodeJS.ReadableStream);
  
  // Validate image
  const metadata = await Sharp(originalBuffer).metadata();
  if (!metadata.width || metadata.width < 100) {
    throw new Error('Image too small');
  }
  
  // Process each variant
  const variants: Record<string, string> = {};
  
  await Promise.all(
    VARIANTS.map(async (variant) => {
      const processed = await Sharp(originalBuffer)
        .resize(variant.width, variant.height, {
          fit: 'inside',
          withoutEnlargement: true,
        })
        .toFormat(variant.format, { quality: variant.quality })
        .toBuffer();
      
      const variantKey = s3Key.replace(/\.[^.]+$/, `-${variant.name}.${variant.format}`);
      
      await s3.send(new PutObjectCommand({
        Bucket: process.env.S3_BUCKET!,
        Key: variantKey,
        Body: processed,
        ContentType: `image/${variant.format}`,
        CacheControl: 'public, max-age=31536000, immutable',
      }));
      
      variants[variant.name] = `${process.env.CDN_URL}/${variantKey}`;
    })
  );
  
  // Update DB with processed variants
  await prisma.uploadedFile.update({
    where: { id: fileId },
    data: {
      processedAt: new Date(),
      variants: variants,
      thumbnailUrl: variants.thumbnail,
    },
  });
  
  return variants;
}
```

---

## 📌 Step 1944: สรุป Part 076

### เนื้อหาที่เรียนรู้

✅ Upload strategies comparison  
✅ Presigned URL pattern  
✅ S3 presigned POST  
✅ File validation & security  
✅ Image processing pipeline  
✅ CDN distribution  

### Upload Security Checklist

```
Validation (before upload):
□ Content-Type whitelist
□ File size limits
□ Extension validation

Validation (after upload):
□ Magic bytes check (ไม่ trust Content-Type)
□ Image dimensions check
□ Malware scanning (optional: ClamAV)
□ Exif data strip (privacy)

Storage:
□ Random key (ไม่ใช่ user-controlled)
□ User prefix isolation
□ Signed URLs สำหรับ private files
□ Expiring upload URLs
□ Cleanup orphaned pending uploads
```

### ในส่วนถัดไป

➡️ **[Part 077](./part-077.md)** — GraphQL Internationalization (i18n)

---

*Part 076 of 100+ | หลักสูตร GraphQL ฉบับสมบูรณ์*
