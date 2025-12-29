# NextJS Frontend - GitHub CMS Integration

> Fetch content từ GitHub và render blog với NextJS

## 🎯 Overview

```
GitHub CMS (content)
         ↓ GitHub API
NextJS App
         ↓ Build/Deploy
Live Website
```

**Approach:** Static Site Generation (SSG) với Incremental Static Regeneration (ISR)

---

## 🚀 Quick Setup (15 phút)

### Bước 1: Create NextJS App

```bash
npx create-next-app@latest my-blog
cd my-blog
```

**Options:**
- TypeScript: Yes
- ESLint: Yes
- Tailwind CSS: Yes (recommended)
- App Router: Yes
- Src directory: No

### Bước 2: Install Dependencies

```bash
npm install octokit gray-matter remark remark-html
```

**Packages:**
- `octokit`: GitHub API client
- `gray-matter`: Parse frontmatter
- `remark` + `remark-html`: Markdown → HTML

### Bước 3: Environment Variables

Create `.env.local`:

```bash
GITHUB_TOKEN=ghp_your_token_here
GITHUB_OWNER=your-username
GITHUB_REPO=my-blog-cms
GITHUB_BRANCH=main
```

**Done!** Ready to code.

---

## 📁 Project Structure

```
my-blog/
├── app/
│   ├── blog/
│   │   ├── [slug]/
│   │   │   └── page.tsx      # Post page
│   │   └── page.tsx          # Blog listing
│   ├── layout.tsx
│   └── page.tsx              # Homepage
├── lib/
│   ├── github.ts             # GitHub API
│   └── posts.ts              # Content helpers
└── .env.local
```

---

## 🔧 Implementation

### 1. GitHub API Client

**File:** `lib/github.ts`

```typescript
import { Octokit } from 'octokit';

const octokit = new Octokit({
  auth: process.env.GITHUB_TOKEN
});

const OWNER = process.env.GITHUB_OWNER!;
const REPO = process.env.GITHUB_REPO!;
const BRANCH = process.env.GITHUB_BRANCH || 'main';

// Get file content
export async function getFileContent(path: string): Promise<string> {
  const { data } = await octokit.rest.repos.getContent({
    owner: OWNER,
    repo: REPO,
    path,
    ref: BRANCH,
  });

  if ('content' in data) {
    return Buffer.from(data.content, 'base64').toString('utf-8');
  }
  throw new Error('Not a file');
}

// List files in directory
export async function listFiles(path: string): Promise<string[]> {
  const { data } = await octokit.rest.repos.getContent({
    owner: OWNER,
    repo: REPO,
    path,
    ref: BRANCH,
  });

  if (Array.isArray(data)) {
    return data
      .filter(item => item.type === 'file' && item.name.endsWith('.md'))
      .map(item => item.name);
  }
  return [];
}
```

### 2. Content Parser

**File:** `lib/posts.ts`

```typescript
import matter from 'gray-matter';
import { remark } from 'remark';
import html from 'remark-html';
import { getFileContent, listFiles } from './github';

export interface Post {
  slug: string;
  title: string;
  description: string;
  publishedAt: string;
  status: string;
  category?: string;
  tags?: string[];
  featuredImage?: string;
  content: string;
  htmlContent?: string;
}

// Get all posts
export async function getAllPosts(): Promise<Post[]> {
  const files = await listFiles('content/posts');
  
  const posts = await Promise.all(
    files.map(async (file) => {
      const slug = file.replace('.md', '');
      return getPostBySlug(slug);
    })
  );

  // Filter published only
  return posts
    .filter(post => post.status === 'published')
    .sort((a, b) => 
      new Date(b.publishedAt).getTime() - new Date(a.publishedAt).getTime()
    );
}

// Get single post
export async function getPostBySlug(slug: string): Promise<Post> {
  const fileContent = await getFileContent(`content/posts/${slug}.md`);
  const { data, content } = matter(fileContent);
  
  // Convert markdown to HTML
  const processedContent = await remark()
    .use(html)
    .process(content);
  
  return {
    slug,
    title: data.title,
    description: data.description,
    publishedAt: data.publishedAt,
    status: data.status,
    category: data.category,
    tags: data.tags,
    featuredImage: data.featuredImage,
    content,
    htmlContent: processedContent.toString(),
  };
}
```

### 3. Blog Listing Page

**File:** `app/blog/page.tsx`

```typescript
import Link from 'next/link';
import Image from 'next/image';
import { getAllPosts } from '@/lib/posts';

export const revalidate = 60; // Revalidate every 60s

export default async function BlogPage() {
  const posts = await getAllPosts();

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-4xl font-bold mb-8">Blog</h1>
      
      <div className="grid gap-8 md:grid-cols-2 lg:grid-cols-3">
        {posts.map((post) => (
          <article 
            key={post.slug} 
            className="border rounded-lg overflow-hidden hover:shadow-lg transition"
          >
            {post.featuredImage && (
              <Link href={`/blog/${post.slug}`}>
                <Image
                  src={post.featuredImage}
                  alt={post.title}
                  width={400}
                  height={250}
                  className="w-full h-48 object-cover"
                />
              </Link>
            )}
            
            <div className="p-4">
              <h2 className="text-2xl font-bold mb-2">
                <Link 
                  href={`/blog/${post.slug}`}
                  className="hover:text-blue-600"
                >
                  {post.title}
                </Link>
              </h2>
              
              <p className="text-gray-600 mb-4">{post.description}</p>
              
              <div className="text-sm text-gray-500">
                {new Date(post.publishedAt).toLocaleDateString('vi-VN')}
              </div>
            </div>
          </article>
        ))}
      </div>
    </div>
  );
}
```

### 4. Single Post Page

**File:** `app/blog/[slug]/page.tsx`

```typescript
import { notFound } from 'next/navigation';
import Image from 'next/image';
import { getAllPosts, getPostBySlug } from '@/lib/posts';

export const revalidate = 60;

// Generate static paths
export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map((post) => ({
    slug: post.slug,
  }));
}

// Generate metadata
export async function generateMetadata({ params }: { params: { slug: string } }) {
  const post = await getPostBySlug(params.slug);
  
  return {
    title: post.title,
    description: post.description,
  };
}

export default async function PostPage({ params }: { params: { slug: string } }) {
  let post;
  
  try {
    post = await getPostBySlug(params.slug);
  } catch {
    notFound();
  }

  if (post.status !== 'published') {
    notFound();
  }

  return (
    <article className="container mx-auto px-4 py-8 max-w-4xl">
      {/* Header */}
      <header className="mb-8">
        <h1 className="text-4xl md:text-5xl font-bold mb-4">
          {post.title}
        </h1>
        
        <div className="text-gray-600">
          {new Date(post.publishedAt).toLocaleDateString('vi-VN', {
            year: 'numeric',
            month: 'long',
            day: 'numeric',
          })}
        </div>
        
        {post.tags && post.tags.length > 0 && (
          <div className="mt-4 flex flex-wrap gap-2">
            {post.tags.map((tag) => (
              <span 
                key={tag}
                className="px-3 py-1 bg-gray-100 rounded text-sm"
              >
                #{tag}
              </span>
            ))}
          </div>
        )}
      </header>

      {/* Featured Image */}
      {post.featuredImage && (
        <div className="mb-8">
          <Image
            src={post.featuredImage}
            alt={post.title}
            width={1200}
            height={630}
            className="w-full rounded-lg"
            priority
          />
        </div>
      )}

      {/* Content */}
      <div
        className="prose prose-lg max-w-none"
        dangerouslySetInnerHTML={{ __html: post.htmlContent || '' }}
      />
    </article>
  );
}
```

---

## ⚡ Optimization

### Image Optimization

**next.config.js:**

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    domains: ['raw.githubusercontent.com'],
    formats: ['image/avif', 'image/webp'],
  },
}

module.exports = nextConfig
```

### Caching Strategy

**ISR (Incremental Static Regeneration):**

```typescript
export const revalidate = 60; // Revalidate every 60 seconds
```

**Options:**
- `60`: Revalidate every minute
- `3600`: Every hour
- `false`: Never revalidate (pure static)

### Error Handling

```typescript
try {
  const post = await getPostBySlug(params.slug);
} catch (error) {
  console.error('Error fetching post:', error);
  notFound();
}
```

---

## 🎨 Styling with Tailwind

### Install Tailwind Typography

```bash
npm install @tailwindcss/typography
```

**tailwind.config.js:**

```javascript
module.exports = {
  plugins: [
    require('@tailwindcss/typography'),
  ],
}
```

**Use in components:**

```tsx
<div className="prose prose-lg max-w-none">
  {/* Markdown content */}
</div>
```

---

## 🚀 Deployment

### Deploy to Vercel (Recommended)

```bash
npm install -g vercel
vercel
```

**Environment variables:**
1. Go to Vercel dashboard
2. Settings → Environment Variables
3. Add:
   - `GITHUB_TOKEN`
   - `GITHUB_OWNER`
   - `GITHUB_REPO`

### Auto-deploy on Git Push

**Connect GitHub repo:**
1. Vercel dashboard → Import Project
2. Select your NextJS repo
3. Configure environment variables
4. Deploy

**Result:** Auto-deploy on every push!

---

## 🔄 Content Update Workflow

### Automatic Revalidation

**With ISR:**
```
1. Edit content on GitHub
2. Commit changes
3. Wait for revalidate interval (e.g., 60s)
4. Content automatically updates
```

### Manual Revalidation (Optional)

**GitHub Webhook → NextJS API:**

**File:** `app/api/revalidate/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { revalidatePath } from 'next/cache';

export async function POST(request: NextRequest) {
  // Verify webhook secret
  const secret = request.headers.get('x-webhook-secret');
  
  if (secret !== process.env.WEBHOOK_SECRET) {
    return NextResponse.json({ message: 'Unauthorized' }, { status: 401 });
  }

  // Revalidate all blog pages
  revalidatePath('/blog');
  
  return NextResponse.json({ revalidated: true });
}
```

**Setup GitHub Webhook:**
1. Repository → Settings → Webhooks
2. Payload URL: `https://yourdomain.com/api/revalidate`
3. Secret: Same as `WEBHOOK_SECRET`
4. Events: `push`

---

## 📊 Advanced Features (Optional)

### Search

```typescript
export async function searchPosts(query: string): Promise<Post[]> {
  const posts = await getAllPosts();
  
  return posts.filter(post =>
    post.title.toLowerCase().includes(query.toLowerCase()) ||
    post.description.toLowerCase().includes(query.toLowerCase())
  );
}
```

### Categories

```typescript
export async function getPostsByCategory(category: string): Promise<Post[]> {
  const posts = await getAllPosts();
  return posts.filter(post => post.category === category);
}
```

### Tags

```typescript
export async function getPostsByTag(tag: string): Promise<Post[]> {
  const posts = await getAllPosts();
  return posts.filter(post => post.tags?.includes(tag));
}
```

### Pagination

```typescript
export async function getPaginatedPosts(
  page: number = 1,
  pageSize: number = 10
): Promise<{ posts: Post[], total: number }> {
  const allPosts = await getAllPosts();
  const start = (page - 1) * pageSize;
  const posts = allPosts.slice(start, start + pageSize);
  
  return { posts, total: allPosts.length };
}
```

---

## 🎯 Performance Checklist

### Build Optimization

- [ ] Enable ISR with appropriate revalidate time
- [ ] Use Next.js Image component
- [ ] Implement lazy loading
- [ ] Optimize image sizes
- [ ] Minify CSS/JS (automatic)

### Runtime Optimization

- [ ] Cache GitHub API responses
- [ ] Implement error boundaries
- [ ] Add loading states
- [ ] Use React.memo for heavy components

---

## 🐛 Troubleshooting

### API Rate Limiting

**Problem:** GitHub API rate limit exceeded

**Solution:**
```typescript
// Add retry logic
async function getFileContentWithRetry(path: string, retries = 3) {
  try {
    return await getFileContent(path);
  } catch (error) {
    if (retries > 0 && error.status === 403) {
      await new Promise(resolve => setTimeout(resolve, 60000));
      return getFileContentWithRetry(path, retries - 1);
    }
    throw error;
  }
}
```

### Build Failures

**Problem:** Build fails during static generation

**Solution:**
```typescript
// Handle errors gracefully
export async function generateStaticParams() {
  try {
    const posts = await getAllPosts();
    return posts.map(post => ({ slug: post.slug }));
  } catch (error) {
    console.error('Error generating static params:', error);
    return [];
  }
}
```

### Images Not Loading

**Problem:** GitHub raw images blocked

**Solution:** Use Next.js Image domains config:
```javascript
images: {
  remotePatterns: [
    {
      protocol: 'https',
      hostname: 'raw.githubusercontent.com',
    },
  ],
}
```

---

## 📈 Monitoring

### Vercel Analytics

```bash
npm install @vercel/analytics
```

**_app.tsx or layout.tsx:**

```typescript
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  );
}
```

### Error Tracking (Optional)

**Sentry:**

```bash
npm install @sentry/nextjs
```

---

## ✅ Summary

**Setup time:** 15 phút
**Build time:** 1-2 phút
**Deploy time:** 3-5 phút

**Total:** < 30 phút from zero to live blog!

**Tech Stack:**
- NextJS 14 (App Router)
- TypeScript
- Tailwind CSS
- GitHub API
- Vercel (hosting)

**Cost:** $0 (Vercel free tier)

**Result:** 
- ⚡ Fast (Static)
- 🔒 Secure
- 📱 Responsive
- 🎨 Beautiful
- 🚀 Scalable

---

## 📚 Complete Example

**Check:**
- `CMS_BACKEND.md` - Content management
- `NEXTJS_FRONTEND.md` - This document

**Together:** Complete blog system!

**Next:** Deploy and start writing! 🎉

---

## 🔗 Resources

- **Next.js Docs:** https://nextjs.org/docs
- **GitHub API:** https://docs.github.com/rest
- **Tailwind CSS:** https://tailwindcss.com
- **Vercel:** https://vercel.com

---

**That's it!** Simple, fast, free blog với GitHub + NextJS. 🚀
