# Next.js Interview Questions

## Overview

Next.js questions test your understanding of server-side rendering, static generation, routing, API routes, and deployment. Essential for full-stack React positions.

---

## Core Next.js Questions

### Q1: Explain the difference between SSR, SSG, and ISR in Next.js

**Answer:**

Next.js offers three main rendering strategies, each with different use cases.

**1. Static Site Generation (SSG):**

Pages generated at build time and reused on every request.

```javascript
// pages/blog/[slug].js
export async function getStaticProps({ params }) {
  const post = await fetchPost(params.slug);
  
  return {
    props: { post }
  };
}

export async function getStaticPaths() {
  const posts = await fetchAllPosts();
  
  return {
    paths: posts.map(post => ({
      params: { slug: post.slug }
    })),
    fallback: false
  };
}

function BlogPost({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}

export default BlogPost;

// Build time: Generates HTML for all posts
// Request time: Serves pre-generated HTML (FAST)
// Use for: Blogs, marketing pages, documentation
```

**2. Server-Side Rendering (SSR):**

Pages generated on every request.

```javascript
// pages/dashboard.js
export async function getServerSideProps(context) {
  const session = await getSession(context);
  
  if (!session) {
    return {
      redirect: {
        destination: '/login',
        permanent: false
      }
    };
  }
  
  const userData = await fetchUserData(session.userId);
  
  return {
    props: { userData }
  };
}

function Dashboard({ userData }) {
  return (
    <div>
      <h1>Welcome, {userData.name}</h1>
      <Stats data={userData.stats} />
    </div>
  );
}

export default Dashboard;

// Request time: Fetches data and generates HTML
// Use for: Personalized pages, dashboards, authenticated content
```

**3. Incremental Static Regeneration (ISR):**

Static pages that update on-demand after a time interval.

```javascript
// pages/products/[id].js
export async function getStaticProps({ params }) {
  const product = await fetchProduct(params.id);
  
  return {
    props: { product },
    revalidate: 60 // Regenerate every 60 seconds
  };
}

export async function getStaticPaths() {
  return {
    paths: [],
    fallback: 'blocking' // Generate on first request
  };
}

function ProductPage({ product }) {
  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      <p>Stock: {product.stock}</p>
    </div>
  );
}

export default ProductPage;

// Build time: Generates initial HTML
// Request time (< 60s): Serves cached HTML (fast)
// Request time (> 60s): Serves cached HTML, regenerates in background
// Next request: Gets updated HTML
// Use for: E-commerce, news sites, frequently updated content
```

**Comparison Table:**

| Strategy | When Generated | Speed | Use Case |
|----------|---------------|-------|----------|
| SSG | Build time | Fastest | Blogs, marketing |
| ISR | Build + periodic | Fast | E-commerce, news |
| SSR | Every request | Slower | Dashboards, auth |
| CSR | Client-side | Initial slow | SPAs, admin panels |

**Fallback Options:**

```javascript
// fallback: false
getStaticPaths() {
  return {
    paths: ['/posts/1', '/posts/2'],
    fallback: false // 404 for any other path
  };
}

// fallback: true (shows loading, generates in background)
getStaticPaths() {
  return {
    paths: [], // Generate none at build
    fallback: true // Generate on demand
  };
}

function Post({ post }) {
  const router = useRouter();
  
  if (router.isFallback) {
    return <div>Loading...</div>;
  }
  
  return <div>{post.title}</div>;
}

// fallback: 'blocking' (waits for generation)
getStaticPaths() {
  return {
    paths: [],
    fallback: 'blocking' // User waits, no loading state needed
  };
}
```

**What Interviewers Look For:**
- Clear understanding of each strategy
- Ability to choose appropriate strategy
- Knowledge of fallback options
- Awareness of revalidate parameter

---

### Q2: Explain Next.js 13+ App Router and Server Components

**Answer:**

App Router is a new paradigm with server components, streaming, and better data fetching.

**Pages Router (Old):**

```javascript
// pages/posts/[id].js
export async function getServerSideProps({ params }) {
  const post = await fetchPost(params.id);
  return { props: { post } };
}

export default function Post({ post }) {
  return <div>{post.title}</div>;
}
```

**App Router (New):**

```javascript
// app/posts/[id]/page.js
async function Post({ params }) {
  const post = await fetchPost(params.id);
  
  return <div>{post.title}</div>;
}

export default Post;

// Server Component by default!
// No getServerSideProps needed
// Can directly use async/await
```

**Server vs Client Components:**

```javascript
// app/dashboard/page.js (Server Component)
async function Dashboard() {
  // Direct database access
  const posts = await db.posts.findMany();
  
  return (
    <div>
      <h1>Dashboard</h1>
      <PostList posts={posts} />
      
      {/* Client component for interactivity */}
      <LikeButton />
    </div>
  );
}

// app/dashboard/LikeButton.js (Client Component)
'use client';

import { useState } from 'react';

export function LikeButton() {
  const [liked, setLiked] = useState(false);
  
  return (
    <button onClick={() => setLiked(!liked)}>
      {liked ? 'Unlike' : 'Like'}
    </button>
  );
}
```

**Layouts and Nested Routing:**

```javascript
// app/layout.js (Root layout)
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Header />
        {children}
        <Footer />
      </body>
    </html>
  );
}

// app/dashboard/layout.js (Nested layout)
export default function DashboardLayout({ children }) {
  return (
    <div>
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}

// app/dashboard/page.js
export default function DashboardPage() {
  return <div>Dashboard Content</div>;
}

// Renders: RootLayout → DashboardLayout → DashboardPage
```

**Loading and Error States:**

```javascript
// app/dashboard/loading.js
export default function Loading() {
  return <Spinner />;
}

// app/dashboard/error.js
'use client';

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// Automatically wrapped with Suspense and ErrorBoundary
```

**Data Fetching Patterns:**

```javascript
// Parallel data fetching
async function Dashboard() {
  const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments()
  ]);
  
  return (
    <div>
      <UserInfo user={user} />
      <PostList posts={posts} />
      <CommentList comments={comments} />
    </div>
  );
}

// Sequential data fetching (when needed)
async function PostPage({ params }) {
  const post = await fetchPost(params.id);
  const author = await fetchAuthor(post.authorId);
  
  return (
    <div>
      <h1>{post.title}</h1>
      <Author author={author} />
    </div>
  );
}

// Streaming with Suspense
async function Dashboard() {
  return (
    <div>
      <UserInfo />
      
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>
      
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments />
      </Suspense>
    </div>
  );
}
```

**What Interviewers Look For:**
- Understanding of App Router benefits
- Knowledge of server/client components
- Familiarity with new file conventions
- Awareness of streaming and suspense

---

### Q3: How do API Routes work in Next.js?

**Answer:**

API routes let you build backend endpoints within your Next.js app.

**Basic API Route:**

```javascript
// pages/api/users.js
export default function handler(req, res) {
  if (req.method === 'GET') {
    const users = [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ];
    
    res.status(200).json(users);
  } else {
    res.status(405).json({ error: 'Method not allowed' });
  }
}

// Access at: /api/users
```

**With Database:**

```javascript
// pages/api/posts/[id].js
import db from '@/lib/db';

export default async function handler(req, res) {
  const { id } = req.query;
  
  if (req.method === 'GET') {
    try {
      const post = await db.post.findUnique({
        where: { id: parseInt(id) }
      });
      
      if (!post) {
        return res.status(404).json({ error: 'Post not found' });
      }
      
      res.status(200).json(post);
    } catch (error) {
      res.status(500).json({ error: 'Database error' });
    }
  } else if (req.method === 'PUT') {
    const { title, content } = req.body;
    
    const updated = await db.post.update({
      where: { id: parseInt(id) },
      data: { title, content }
    });
    
    res.status(200).json(updated);
  } else if (req.method === 'DELETE') {
    await db.post.delete({
      where: { id: parseInt(id) }
    });
    
    res.status(204).end();
  } else {
    res.status(405).json({ error: 'Method not allowed' });
  }
}
```

**Authentication:**

```javascript
// pages/api/protected.js
import { getSession } from 'next-auth/react';

export default async function handler(req, res) {
  const session = await getSession({ req });
  
  if (!session) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  // User is authenticated
  const data = await fetchUserData(session.user.id);
  res.status(200).json(data);
}
```

**Middleware:**

```javascript
// middleware.js
import { NextResponse } from 'next/server';

export function middleware(request) {
  // Add custom header
  const response = NextResponse.next();
  response.headers.set('x-custom-header', 'value');
  
  // Redirect based on condition
  if (request.nextUrl.pathname === '/old-path') {
    return NextResponse.redirect(new URL('/new-path', request.url));
  }
  
  // Rewrite
  if (request.nextUrl.pathname.startsWith('/blog')) {
    return NextResponse.rewrite(new URL('/news', request.url));
  }
  
  return response;
}

export const config = {
  matcher: '/api/:path*'
};
```

**App Router Route Handlers:**

```javascript
// app/api/users/route.js
import { NextResponse } from 'next/server';

export async function GET(request) {
  const users = await db.users.findMany();
  return NextResponse.json(users);
}

export async function POST(request) {
  const body = await request.json();
  const user = await db.users.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}

// app/api/users/[id]/route.js
export async function GET(request, { params }) {
  const user = await db.users.findUnique({
    where: { id: params.id }
  });
  
  if (!user) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }
  
  return NextResponse.json(user);
}
```

**What Interviewers Look For:**
- Understanding of API route structure
- Knowledge of HTTP methods
- Error handling
- Authentication patterns
- App Router route handlers

---

### Q4: How do you optimize images and fonts in Next.js?

**Answer:**

Next.js provides built-in optimization for images and fonts.

**Image Optimization:**

```javascript
import Image from 'next/image';

// Optimized image
function Profile() {
  return (
    <Image
      src="/profile.jpg"
      alt="Profile"
      width={500}
      height={500}
      priority // Load immediately (above fold)
    />
  );
}

// Remote images
function RemoteImage() {
  return (
    <Image
      src="https://example.com/image.jpg"
      alt="Remote"
      width={800}
      height={600}
      // Must configure domains in next.config.js
    />
  );
}

// next.config.js
module.exports = {
  images: {
    domains: ['example.com'],
    formats: ['image/avif', 'image/webp']
  }
};

// Responsive images
function ResponsiveImage() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero"
      fill
      style={{ objectFit: 'cover' }}
      sizes="(max-width: 768px) 100vw, 50vw"
    />
  );
}

// Lazy loading (default)
function Gallery({ images }) {
  return images.map((img, i) => (
    <Image
      key={i}
      src={img.src}
      alt={img.alt}
      width={300}
      height={200}
      loading="lazy" // Default behavior
    />
  ));
}
```

**Font Optimization:**

```javascript
// app/layout.js
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap'
});

const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  weight: ['400', '700'],
  display: 'swap'
});

export default function RootLayout({ children }) {
  return (
    <html className={inter.className}>
      <body>
        {children}
        <code className={robotoMono.className}>Code block</code>
      </body>
    </html>
  );
}

// Custom fonts
import localFont from 'next/font/local';

const myFont = localFont({
  src: './my-font.woff2',
  display: 'swap'
});
```

**What Interviewers Look For:**
- Using Next.js Image component
- Understanding of image optimization
- Font loading strategies
- Performance considerations

---

### Q5: How do you deploy and configure Next.js apps?

**Answer:**

**Vercel Deployment (Easiest):**

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel

# Production
vercel --prod
```

**Environment Variables:**

```javascript
// .env.local
DATABASE_URL=postgres://...
NEXT_PUBLIC_API_URL=https://api.example.com

// Access in server components/API routes
process.env.DATABASE_URL

// Access in client components (must start with NEXT_PUBLIC_)
process.env.NEXT_PUBLIC_API_URL
```

**next.config.js Configuration:**

```javascript
// next.config.js
module.exports = {
  // React strict mode
  reactStrictMode: true,
  
  // Image domains
  images: {
    domains: ['cdn.example.com']
  },
  
  // Redirects
  async redirects() {
    return [
      {
        source: '/old-path',
        destination: '/new-path',
        permanent: true
      }
    ];
  },
  
  // Rewrites (proxy)
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://api.example.com/:path*'
      }
    ];
  },
  
  // Headers
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY'
          }
        ]
      }
    ];
  }
};
```

**Docker Deployment:**

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

**What Interviewers Look For:**
- Deployment experience
- Environment variable management
- Configuration knowledge
- Production best practices

---

## Key Takeaways

**Essential Concepts:**
1. SSG for static content, SSR for dynamic
2. ISR for periodic updates
3. App Router with server components
4. API routes for backend logic
5. Built-in image and font optimization

**Best Practices:**
- Choose appropriate rendering strategy
- Use Server Components by default
- Optimize images with Next/Image
- Configure environment variables properly
- Use ISR for frequently updated content

**Common Mistakes:**
- Using SSR when SSG would work
- Not optimizing images
- Forgetting NEXT_PUBLIC_ prefix
- Over-using client components
- Ignoring middleware capabilities

**Interview Tips:**
- Compare rendering strategies
- Show App Router knowledge
- Demonstrate API route usage
- Discuss optimization techniques
- Explain deployment process

**Red Flags to Avoid:**
- Only knowing Pages Router
- Not understanding SSR vs SSG
- Never deployed to production
- Ignoring performance optimization
- Not familiar with Server Components
