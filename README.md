# Tutorial: Creating an Angular 19 Standalone Blog with Global Styles, Update Functionality, Separate HTML Files, CloudFront, S3, API Gateway, Lambda, and DynamoDB

This tutorial guides you through building a **standalone Angular 19 blog application** with separate `.html` template files, a comprehensive global `styles.css` for a dark theme, and full CRUD functionality, including an `updatePost` method in `BlogService`. It resolves the following issues:

- **TS2305: Module '@angular/forms' has no exported member 'provideForms'**: Fixed by removing `provideForms` from `app.config.ts` and importing `ReactiveFormsModule` in `AdminComponent`.
- **Navigation links not working**: Fixed by adding `RouterLink` to `AppComponent`’s `imports` array.
- **CORS Missing Allow Origin**: Fixed by enabling CORS for `GET /posts/{postId}` and `OPTIONS /posts/{postId}` in API Gateway, ensuring `Access-Control-Allow-Origin: *` in responses.

The app is hosted on **Amazon S3** with **CloudFront** for low-latency delivery, using **Origin Access Control (OAC)** to secure access and address the “Create Control Setting” issue. It integrates with **API Gateway**, **Lambda** (`createPost`, `readPosts`, `getPost`, `updatePost`, `deletePost` with CORS headers), and **DynamoDB** for CRUD operations, resolving errors like "Missing Authentication Token" and "CORS Missing Allow Origin" for the API endpoint (`https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod`). The guide aligns with your setup (`main.ts`, `app.config.ts`, `app.routes.ts`, styled components, reactive forms) and leverages the AWS free tier. It excludes zipped source files and school file storage.

## Prerequisites
- **Node.js**: Version 18.x or later (Angular 19 requirement).
- **Angular CLI**: Install globally: `npm install -g @angular/cli@19`.
- **AWS Account**: Sign up at [aws.amazon.com](https://aws.amazon.com) (free tier: 1 TB CloudFront egress, 1,000 invalidations, 5 GB S3 Standard, 2,000 PUT requests, 1M API Gateway calls, 25 GB DynamoDB, 1M Lambda requests).
- **IAM Permissions**: Ensure your IAM user/role has permissions for:
  - `cloudfront:CreateOriginAccessControl`, `cloudfront:UpdateDistribution`, `cloudfront:GetOriginAccessControl`
  - `s3:PutBucketPolicy`, `s3:GetBucketPolicy`, `s3:PutObject`, `s3:ListBucket`
  - `dynamodb:PutItem`, `dynamodb:Scan`, `dynamodb:GetItem`, `dynamodb:UpdateItem`, `dynamodb:DeleteItem`
  - `lambda:CreateFunction`, `lambda:UpdateFunctionCode`, `lambda:InvokeFunction`
  - `apigateway:*`
  - Check via **IAM** → **Users** → Your user → **Permissions**.
- **AWS CLI**: Installed and configured (`aws configure`).
- **Text Editor**: VS Code or similar.
- **Git**: Optional for version control.

## Step 1: Create the Angular 19 Standalone Project
1. **Create Project**:
   - Run:
     ```bash
     ng new blog-app --standalone --style=css --routing
     ```
   - Options:
     - `--standalone`: Creates a standalone project (no NgModules).
     - `--style=css`: Uses CSS, creating `src/styles.css` for global styles.
     - `--routing`: Adds `app.routes.ts` for routing.
   - Navigate: `cd blog-app`.
2. **Install Dependencies**:
   - Dependencies like `@angular/core`, `@angular/router`, `@angular/common`, `@angular/forms` are included by default.
   - Verify: `npm install`.
3. **Set Up Project Structure**:
   - Create components and service:
     ```bash
     ng generate component home
     ng generate component blog
     ng generate component post-detail
     ng generate component admin
     ng generate service blog
     ```
   - Resulting structure:
     ```
     blog-app/
     ├── src/
     │   ├── app/
     │   │   ├── home/
     │   │   │   ├── home.component.ts
     │   │   │   ├── home.component.html
     │   │   │   └── home.component.css
     │   │   ├── blog/
     │   │   │   ├── blog.component.ts
     │   │   │   ├── blog.component.html
     │   │   │   └── blog.component.css
     │   │   ├── post-detail/
     │   │   │   ├── post-detail.component.ts
     │   │   │   ├── post-detail.component.html
     │   │   │   └── post-detail.component.css
     │   │   ├── admin/
     │   │   │   ├── admin.component.ts
     │   │   │   ├── admin.component.html
     │   │   │   └── admin.component.css
     │   │   ├── blog.service.ts
     │   │   ├── app.component.ts
     │   │   ├── app.component.html
     │   │   ├── app.component.css
     │   │   ├── app.config.ts
     │   │   ├── app.routes.ts
     │   │   └── main.ts
     │   ├── assets/
     │   └── styles.css
     ├── angular.json
     ├── package.json
     └── ...
     ```
   - **Note**: `styles.css` is included in `angular.json`:
     ```json
     "projects": {
       "blog-app": {
         "architect": {
           "build": {
             "options": {
               "styles": ["src/styles.css"]
             }
           }
         }
       }
     }
     ```
4. **Configure Global Styles** (`styles.css`):
   - Update `src/styles.css`:
     ```css
     * {
       margin: 0;
       padding: 0;
       box-sizing: border-box;
     }

     body {
       font-family: 'Arial', sans-serif;
       background-color: #1a1a1a;
       color: #e0e0e0;
       line-height: 1.6;
     }

     a {
       color: #007bff;
       text-decoration: none;
     }

     a:hover {
       color: #0056b3;
     }

     .container {
       max-width: 1200px;
       margin: 0 auto;
       padding: 20px;
     }

     nav {
       background-color: #2c2c2c;
       padding: 10px 20px;
       position: fixed;
       top: 0;
       width: 100%;
       z-index: 1000;
       box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
     }

     nav a {
       margin-right: 20px;
       font-weight: bold;
     }

     nav a:hover {
       color: #ffffff;
     }

     @media (max-width: 600px) {
       nav {
         padding: 10px;
       }
       nav a {
         margin-right: 10px;
         font-size: 14px;
       }
       .container {
         padding: 10px;
       }
     }
     ```
5. **Configure Routing** (`app.routes.ts`):
   - Update `src/app/app.routes.ts`:
     ```typescript
     import { Routes } from '@angular/router';
     import { HomeComponent } from './home/home.component';
     import { BlogComponent } from './blog/blog.component';
     import { PostDetailComponent } from './post-detail/post-detail.component';
     import { AdminComponent } from './admin/admin.component';

     export const routes: Routes = [
       { path: '', component: HomeComponent },
       { path: 'blog', component: BlogComponent },
       { path: 'post/:id', component: PostDetailComponent },
       { path: 'admin', component: AdminComponent },
       { path: '**', redirectTo: '' }
     ];
     ```
6. **Configure Providers** (`app.config.ts`):
   - Update `src/app/app.config.ts`:
     ```typescript
     import { ApplicationConfig } from '@angular/core';
     import { provideRouter } from '@angular/router';
     import { provideHttpClient } from '@angular/common/http';
     import { routes } from './app.routes';

     export const appConfig: ApplicationConfig = {
       providers: [
         provideRouter(routes),
         provideHttpClient()
       ]
     };
     ```
7. **Update Main Component**:
   - Update `src/app/app.component.ts`:
     ```typescript
     import { Component } from '@angular/core';
     import { RouterOutlet, RouterLink } from '@angular/router';
     import { CommonModule } from '@angular/common';

     @Component({
       selector: 'app-root',
       standalone: true,
       imports: [RouterOutlet, RouterLink, CommonModule],
       templateUrl: './app.component.html',
       styleUrls: ['./app.component.css']
     })
     export class AppComponent {}
     ```
   - Update `src/app/app.component.html`:
     ```html
     <header>
       <nav>
         <a routerLink="/">Home</a>
         <a routerLink="/blog">Blog</a>
         <a routerLink="/admin">Admin</a>
       </nav>
     </header>
     <div class="container">
       <router-outlet></router-outlet>
     </div>
     ```
   - Update `src/app/app.component.css`:
     ```css
     :host {
       display: block;
       margin-top: 60px;
     }
     ```
8. **Update Blog Service** (`blog.service.ts`):
   - Update `src/app/blog.service.ts`:
     ```typescript
     import { Injectable } from '@angular/core';
     import { HttpClient } from '@angular/common/http';
     import { Observable } from 'rxjs';

     export interface BlogPost {
       postId: string;
       title: string;
       content: string;
       imageUrl?: string;
       createdAt: string;
     }

     @Injectable({ providedIn: 'root' })
     export class BlogService {
       private apiUrl = 'https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts';

       constructor(private http: HttpClient) {}

       createPost(post: BlogPost): Observable<any> {
         return this.http.post(this.apiUrl, post);
       }

       getPosts(): Observable<BlogPost[]> {
         return this.http.get<BlogPost[]>(this.apiUrl);
       }

       getPost(postId: string): Observable<BlogPost> {
         return this.http.get<BlogPost>(`${this.apiUrl}/${postId}`);
       }

       updatePost(postId: string, post: BlogPost): Observable<any> {
         return this.http.put(`${this.apiUrl}/${postId}`, post);
       }

       deletePost(postId: string): Observable<any> {
         return this.http.delete(`${this.apiUrl}/${postId}`);
       }
     }
     ```
9. **Update Components**:
   - **Home**:
     - Update `src/app/home/home.component.ts`:
       ```typescript
       import { Component } from '@angular/core';
       import { CommonModule } from '@angular/common';

       @Component({
         selector: 'app-home',
         standalone: true,
         imports: [CommonModule],
         templateUrl: './home.component.html',
         styleUrls: ['./home.component.css']
       })
       export class HomeComponent {}
       ```
     - Update `src/app/home/home.component.html`:
       ```html
       <section class="container">
         <h1>Welcome to My Blog</h1>
         <p>Explore our posts or manage content in the admin panel.</p>
       </section>
       ```
     - Update `src/app/home/home.component.css`:
       ```css
       h1 {
         color: #e0e0e0;
         text-align: center;
         margin-bottom: 1rem;
       }
       p {
         text-align: center;
       }
       ```
   - **Blog**:
     - Update `src/app/blog/blog.component.ts`:
       ```typescript
       import { Component, OnInit } from '@angular/core';
       import { CommonModule } from '@angular/common';
       import { RouterLink } from '@angular/router';
       import { BlogService, BlogPost } from '../blog.service';

       @Component({
         selector: 'app-blog',
         standalone: true,
         imports: [CommonModule, RouterLink],
         templateUrl: './blog.component.html',
         styleUrls: ['./blog.component.css']
       })
       export class BlogComponent implements OnInit {
         posts: BlogPost[] = [];

         constructor(private blogService: BlogService) {}

         ngOnInit() {
           this.blogService.getPosts().subscribe(posts => {
             this.posts = posts;
           });
         }
       }
       ```
     - Update `src/app/blog/blog.component.html`:
       ```html
       <section class="container">
         <h2>Blog Posts</h2>
         <div class="post-grid">
           <div *ngFor="let post of posts" class="post-card">
             <h3>{{ post.title }}</h3>
             <p>{{ post.content | slice:0:100 }}...</p>
             <a [routerLink]="['/post', post.postId]">Read More</a>
           </div>
         </div>
       </section>
       ```
     - Update `src/app/blog/blog.component.css`:
       ```css
       h2 {
         color: #e0e0e0;
         margin-bottom: 1rem;
       }
       .post-grid {
         display: grid;
         grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
         gap: 1rem;
       }
       .post-card {
         background: #2c2c2c;
         padding: 1rem;
         border-radius: 5px;
         box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
       }
       h3 {
         color: #e0e0e0;
       }
       a {
         color: #007bff;
       }
       a:hover {
         color: #ffffff;
       }
       ```
   - **Post Detail**:
     - Update `src/app/post-detail/post-detail.component.ts`:
       ```typescript
       import { Component, OnInit } from '@angular/core';
       import { CommonModule } from '@angular/common';
       import { ActivatedRoute } from '@angular/router';
       import { BlogService, BlogPost } from '../blog.service';

       @Component({
         selector: 'app-post-detail',
         standalone: true,
         imports: [CommonModule],
         templateUrl: './post-detail.component.html',
         styleUrls: ['./post-detail.component.css']
       })
       export class PostDetailComponent implements OnInit {
         post: BlogPost | null = null;

         constructor(private route: ActivatedRoute, private blogService: BlogService) {}

         ngOnInit() {
           const postId = this.route.snapshot.paramMap.get('id');
           if (postId) {
             this.blogService.getPost(postId).subscribe({
               next: (post) => {
                 this.post = post;
               },
               error: (err) => {
                 console.error('API Error:', err);
                 this.post = null;
               }
             });
           }
         }
       }
       ```
     - Update `src/app/post-detail/post-detail.component.html`:
       ```html
       <section class="container">
         <div *ngIf="post">
           <h2>{{ post.title }}</h2>
           <p>{{ post.content }}</p>
           <img *ngIf="post.imageUrl" [src]="post.imageUrl" alt="{{ post.title }}" class="post-image">
           <p><small>Posted on {{ post.createdAt | date }}</small></p>
         </div>
         <div *ngIf="!post">
           <p>Post not found.</p>
         </div>
       </section>
       ```
     - Update `src/app/post-detail/post-detail.component.css`:
       ```css
       h2 {
         color: #e0e0e0;
         margin-bottom: 1rem;
       }
       .post-image {
         max-width: 100%;
         border-radius: 5px;
         margin: 1rem 0;
       }
       p {
         color: #e0e0e0;
       }
       small {
         color: #a0a0a0;
       }
       ```
   - **Admin**:
     - Update `src/app/admin/admin.component.ts`:
       ```typescript
       import { Component, OnInit } from '@angular/core';
       import { CommonModule } from '@angular/common';
       import { ReactiveFormsModule, FormBuilder, FormGroup, Validators } from '@angular/forms';
       import { BlogService, BlogPost } from '../blog.service';

       @Component({
         selector: 'app-admin',
         standalone: true,
         imports: [CommonModule, ReactiveFormsModule],
         templateUrl: './admin.component.html',
         styleUrls: ['./admin.component.css']
       })
       export class AdminComponent implements OnInit {
         postForm: FormGroup;
         editForm: FormGroup;
         posts: BlogPost[] = [];
         editingPost: BlogPost | null = null;

         constructor(private fb: FormBuilder, private blogService: BlogService) {
           this.postForm = this.fb.group({
             title: ['', Validators.required],
             content: ['', Validators.required],
             imageUrl: ['']
           });
           this.editForm = this.fb.group({
             postId: [''],
             title: ['', Validators.required],
             content: ['', Validators.required],
             imageUrl: [''],
             createdAt: ['']
           });
         }

         ngOnInit() {
           this.loadPosts();
         }

         loadPosts() {
           this.blogService.getPosts().subscribe(posts => {
             this.posts = posts;
           });
         }

         onSubmit() {
           this.blogService.createPost(this.postForm.value).subscribe(() => {
             this.postForm.reset();
             this.loadPosts();
           });
         }

         editPost(post: BlogPost) {
           this.editingPost = post;
           this.editForm.patchValue({
             postId: post.postId,
             title: post.title,
             content: post.content,
             imageUrl: post.imageUrl,
             createdAt: post.createdAt
           });
         }

         onUpdate() {
           if (this.editingPost) {
             this.blogService.updatePost(this.editingPost.postId, this.editForm.value).subscribe(() => {
               this.editingPost = null;
               this.editForm.reset();
               this.loadPosts();
             });
           }
         }

         cancelEdit() {
           this.editingPost = null;
           this.editForm.reset();
         }

         deletePost(postId: string) {
           this.blogService.deletePost(postId).subscribe(() => {
             this.loadPosts();
           });
         }
       }
       ```
     - Update `src/app/admin/admin.component.html`:
       ```html
       <section class="container">
         <h2>Create Post</h2>
         <form [formGroup]="postForm" (ngSubmit)="onSubmit()">
           <div class="form-group">
             <label>Title:</label>
             <input formControlName="title">
           </div>
           <div class="form-group">
             <label>Content:</label>
             <textarea formControlName="content"></textarea>
           </div>
           <div class="form-group">
             <label>Image URL:</label>
             <input formControlName="imageUrl">
           </div>
           <button type="submit" [disabled]="postForm.invalid">Create Post</button>
         </form>

         <h2 *ngIf="editingPost">Edit Post</h2>
         <form *ngIf="editingPost" [formGroup]="editForm" (ngSubmit)="onUpdate()">
           <div class="form-group">
             <label>Title:</label>
             <input formControlName="title">
           </div>
           <div class="form-group">
             <label>Content:</label>
             <textarea formControlName="content"></textarea>
           </div>
           <div class="form-group">
             <label>Image URL:</label>
             <input formControlName="imageUrl">
           </div>
           <button type="submit" [disabled]="editForm.invalid">Update Post</button>
           <button type="button" (click)="cancelEdit()">Cancel</button>
         </form>

         <h2>Posts</h2>
         <ul>
           <li *ngFor="let post of posts">
             {{ post.title }}
             <button (click)="editPost(post)">Edit</button>
             <button (click)="deletePost(post.postId)">Delete</button>
           </li>
         </ul>
       </section>
       ```
     - Update `src/app/admin/admin.component.css`:
       ```css
       h2 {
         color: #e0e0e0;
         margin-bottom: 1rem;
       }
       .form-group {
         margin-bottom: 1rem;
       }
       label {
         display: block;
         color: #e0e0e0;
         margin-bottom: 0.5rem;
       }
       input, textarea {
         width: 100%;
         padding: 0.5rem;
         background: #2c2c2c;
         color: #e0e0e0;
         border: 1px solid #444;
         border-radius: 5px;
       }
       textarea {
         min-height: 100px;
       }
       button {
         padding: 0.5rem 1rem;
         background: #007bff;
         color: #fff;
         border: none;
         border-radius: 5px;
         cursor: pointer;
         margin-right: 0.5rem;
       }
       button:disabled {
         background: #555;
         cursor: not-allowed;
       }
       button[type="button"] {
         background: #6c757d;
       }
       button[type="button"]:hover {
         background: #5a6268;
       }
       ul {
         list-style: none;
         padding: 0;
       }
       li {
         margin-bottom: 0.5rem;
         color: #e0e0e0;
       }
       li button:nth-child(2) {
         background: #dc3545;
       }
       li button:nth-child(2):hover {
         background: #c82333;
       }
       ```
10. **Test Locally**:
    - Run: `ng serve --open`.
    - Verify:
      - Application compiles without `TS2305` error.
      - Navigation (`/`, `/blog`, `/admin`, `/post/1753978262517`) works with dark-themed nav.
      - Global styles (`#1a1a1a` background, `.container`) apply.
      - Admin page allows creating, editing, and deleting posts.
      - Post detail page loads without CORS errors.
      - Responsive design works at `<600px`.

## Step 2: Set Up DynamoDB
1. **Create Table**:
   - Go to **AWS Console** → **DynamoDB** → **Tables** → **Create table**.
   - **Table name**: `BlogPosts`.
   - **Partition key**: `postId` (String).
   - **Billing mode**: Provisioned (1 RCU/WCU, free tier).
   - Click **Create table**.
2. **Verify Schema**:
   - Attributes: `postId` (String), `title` (String), `content` (String), `imageUrl` (String, optional), `createdAt` (String).

## Step 3: Create Lambda Functions
1. **Create IAM Role**:
   - Go to **IAM** → **Roles** → **Create role**.
   - **Trusted entity**: Lambda.
   - **Permissions**:
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": [
             "dynamodb:PutItem",
             "dynamodb:Scan",
             "dynamodb:GetItem",
             "dynamodb:UpdateItem",
             "dynamodb:DeleteItem"
           ],
           "Resource": "arn:aws:dynamodb:us-west-1:371751795928:table/BlogPosts"
         },
         {
           "Effect": "Allow",
           "Action": [
             "logs:CreateLogGroup",
             "logs:CreateLogStream",
             "logs:PutLogEvents"
           ],
           "Resource": "*"
         }
       ]
     }
     ```
   - **Name**: `BlogLambdaRole`.
   - Click **Create role**.
2. **Create Lambda Functions**:
   - Go to **Lambda** → **Functions** → **Create function**.
   - For each function (`createPost`, `readPosts`, `getPost`, `updatePost`, `deletePost`):
     - **Function name**: e.g., `updatePost`.
     - **Runtime**: Python 3.9 or later.
     - **Role**: `BlogLambdaRole`.
     - Click **Create function**.
3. **Add Code**:
   - **createPost**:
     ```python
     import json
     import boto3
     import uuid
     from datetime import datetime

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             body = json.loads(event.get("body", "{}"))
             post_id = str(uuid.uuid4())
             title = body.get("title")
             content = body.get("content")
             image_url = body.get("imageUrl")
             if not title or not content:
                 return {
                     "statusCode": 400,
                     "headers": {
                         "Access-Control-Allow-Origin": "*",
                         "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                         "Access-Control-Allow-Methods": "POST,GET,OPTIONS"
                     },
                     "body": json.dumps({"error": "title and content are required"})
                 }
             table.put_item(Item={
                 "postId": post_id,
                 "title": title,
                 "content": content,
                 "imageUrl": image_url,
                 "createdAt": datetime.utcnow().isoformat()
             })
             return {
                 "statusCode": 201,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "POST,GET,OPTIONS"
                 },
                 "body": json.dumps({"postId": post_id, "message": "Post created"})
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "POST,GET,OPTIONS"
                 },
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **readPosts**:
     ```python
     import json
     import boto3

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             response = table.scan()
             posts = response.get("Items", [])
             return {
                 "statusCode": 200,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "POST,GET,OPTIONS"
                 },
                 "body": json.dumps(posts)
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "POST,GET,OPTIONS"
                 },
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **getPost**:
     ```python
     import json
     import boto3

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             post_id = event.get("pathParameters", {}).get("postId")
             if not post_id:
                 return {
                     "statusCode": 400,
                     "headers": {
                         "Access-Control-Allow-Origin": "*",
                         "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                         "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                     },
                     "body": json.dumps({"error": "postId is required"})
                 }
             response = table.get_item(Key={"postId": post_id})
             post = response.get("Item")
             if not post:
                 return {
                     "statusCode": 404,
                     "headers": {
                         "Access-Control-Allow-Origin": "*",
                         "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                         "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                     },
                     "body": json.dumps({"error": f"Post with postId {post_id} not found"})
                 }
             return {
                 "statusCode": 200,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                 },
                 "body": json.dumps(post)
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                 },
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **updatePost**:
     ```python
     import json
     import boto3

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             post_id = event.get("pathParameters", {}).get("postId")
             body = json.loads(event.get("body", "{}"))
             title = body.get("title")
             content = body.get("content")
             image_url = body.get("imageUrl")
             created_at = body.get("createdAt")

             if not post_id or not title or not content:
                 return {
                     "statusCode": 400,
                     "headers": {
                         "Access-Control-Allow-Origin": "*",
                         "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                         "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                     },
                     "body": json.dumps({"error": "postId, title, and content are required"})
                 }

             response = table.get_item(Key={"postId": post_id})
             if not response.get("Item"):
                 return {
                     "statusCode": 404,
                     "headers": {
                         "Access-Control-Allow-Origin": "*",
                         "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                         "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                     },
                     "body": json.dumps({"error": f"Post with postId {post_id} not found"})
                 }

             update_expression = "SET title = :title, content = :content"
             expression_attribute_values = {
                 ":title": title,
                 ":content": content
             }

             if image_url is not None:
                 update_expression += ", imageUrl = :imageUrl"
                 expression_attribute_values[":imageUrl"] = image_url

             if created_at:
                 update_expression += ", createdAt = :createdAt"
                 expression_attribute_values[":createdAt"] = created_at

             table.update_item(
                 Key={"postId": post_id},
                 UpdateExpression=update_expression,
                 ExpressionAttributeValues=expression_attribute_values
             )

             return {
                 "statusCode": 200,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                 },
                 "body": json.dumps({"message": f"Post {post_id} updated"})
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                 },
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **deletePost**:
     ```python
     import json
     import boto3

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             post_id = event.get("pathParameters", {}).get("postId")
             if not post_id:
                 return {
                     "statusCode": 400,
                     "headers": {
                         "Access-Control-Allow-Origin": "*",
                         "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                         "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                     },
                     "body": json.dumps({"error": "postId is required"})
                 }
             response = table.delete_item(Key={"postId": post_id}, ReturnValues="ALL_OLD")
             if "Attributes" not in response:
                 return {
                     "statusCode": 404,
                     "headers": {
                         "Access-Control-Allow-Origin": "*",
                         "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                         "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                     },
                     "body": json.dumps({"error": f"Post with postId {post_id} not found"})
                 }
             return {
                 "statusCode": 200,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                 },
                 "body": json.dumps({"message": f"Post {post_id} deleted"})
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "headers": {
                     "Access-Control-Allow-Origin": "*",
                     "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                     "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
                 },
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - Click **Deploy** for each function.

## Step 4: Set Up API Gateway
1. **Create API**:
   - Go to **AWS Console** → **API Gateway** → **APIs** → **Create API** → REST API → **Build**.
   - **API name**: `BlogAPI`.
   - **Endpoint Type**: Regional.
   - Click **Create API**.
2. **Create Resources**:
   - **Actions** → **Create Resource** → Name: `posts`, Path: `/posts` → **Create Resource**.
   - Select `/posts` → **Actions** → **Create Resource** → Name: `postId`, Path: `{postId}` → **Create Resource**.
3. **Configure Methods**:
   - **POST /posts**:
     - Select `/posts` → **Actions** → **Create Method** → **POST**.
     - **Integration type**: Lambda Function.
     - **Lambda Region**: `us-west-1`.
     - **Lambda Function**: `createPost`.
     - **Method Request**: Set **Authorization**: `NONE`, **API Key Required**: `false`.
     - **Integration Response** → Enable **Lambda Proxy Integration**.
   - **GET /posts**:
     - Select `/posts` → **Create Method** → **GET**.
     - Integrate with `readPosts` Lambda.
     - Set **Authorization**: `NONE`, **API Key Required**: `false`.
   - **GET /posts/{postId}**:
     - Select `/posts/{postId}` → **Create Method** → **GET**.
     - Integrate with `getPost` Lambda.
     - Set **Authorization**: `NONE`, **API Key Required**: `false`.
     - **Integration Response** → Enable **Lambda Proxy Integration**.
   - **PUT /posts/{postId}**:
     - Select `/posts/{postId}` → **Create Method** → **PUT**.
     - Integrate with `updatePost` Lambda.
     - Set **Authorization**: `NONE`, **API Key Required**: `false`.
   - **DELETE /posts/{postId}**:
     - Select `/posts/{postId}` → **Create Method** → **DELETE**.
     - Integrate with `deletePost` Lambda.
     - Set **Authorization**: `NONE`, **API Key Required**: `false`.
   - **OPTIONS /posts** and **OPTIONS /posts/{postId}**:
     - For each resource, create **OPTIONS** method:
       - **Integration type**: **Mock**.
       - **Integration Response** → Expand **200** → **Response Headers**:
         - `Access-Control-Allow-Origin: *`
         - `Access-Control-Allow-Headers: Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token`
         - `Access-Control-Allow-Methods: POST,GET,PUT,DELETE,OPTIONS`
4. **Enable CORS**:
   - Select `/posts` → **Actions** → **Enable CORS**.
     - **Methods**: Check `POST`, `GET`, `OPTIONS`.
     - **Access-Control-Allow-Headers**: `Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token`
     - **Access-Control-Allow-Origin**: `*` (for testing) or `https://<cloudfront-id>.cloudfront.net`.
     - Click **Enable CORS and replace existing CORS headers**.
   - Repeat for `/posts/{postId}`:
     - **Methods**: Check `GET`, `PUT`, `DELETE`, `OPTIONS`.
     - **Access-Control-Allow-Headers**: `Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token`
     - **Access-Control-Allow-Origin**: `*` (for testing) or `https://<cloudfront-id>.cloudfront.net`.
     - Click **Enable CORS and replace existing CORS headers**.
5. **Deploy API**:
   - **Actions** → **Deploy API** → **Stage**: `prod`.
   - Verify endpoint: `https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod`.

## Step 5: Create S3 Bucket
1. **Create Bucket**:
   - Go to **S3** → **Create bucket**.
   - **Bucket name**: `your-blog-bucket` (unique, e.g., `your-blog-bucket-123`).
   - **Region**: `us-west-1`.
   - **Object Ownership**: **Bucket owner enforced**.
   - **Block Public Access**: Enable **Block all public access**.
   - Click **Create bucket**.
2. **Upload Angular Build**:
   - Run: `ng build --configuration production`.
   - Upload **contents** of `dist/blog-app/` to bucket root:
     - Go to **S3** → `your-blog-bucket` → **Upload**.
     - Drag and drop `index.html`, `main.js`, `styles.css`, `assets/`, etc.
     - Ensure `styles.css` is included and `assets/` retains structure.
     - Upload (~$0.00025 for 50 files, free tier: 2,000 PUTs).
   - Verify structure:
     ```
     s3://your-blog-bucket/
     ├── index.html
     ├── main.js
     ├── polyfills.js
     ├── styles.css
     ├── assets/
     │   ├── images/
     │   ├── favicon.ico
     │   └── ...
     ```

## Step 6: Create CloudFront Distribution with OAC
1. **Access CloudFront**:
   - Go to **AWS Console** → **CloudFront** → **Create Distribution**.
2. **Configure Origin**:
   - **Origin Domain**: Select `your-blog-bucket.s3.amazonaws.com` (REST API endpoint).
   - **Origin Access**:
     - Select **Origin access control settings (recommended)**.
     - If **Create control setting** is visible:
       - Click **Create control setting**.
       - **Name**: `BlogOAC`.
       - **Origin Type**: `S3`.
       - **Signing Behavior**: `Sign requests (recommended)`.
       - **Signing Protocol**: `SigV4`.
       - Click **Create**.
     - If **Create control setting** is missing:
       - Go to **CloudFront** → **Security** → **Origin access** → **Create origin access control**.
       - Use same settings as above.
       - Alternatively, use AWS CLI:
         ```bash
         aws cloudfront create-origin-access-control \
           --origin-access-control-config \
             Name=BlogOAC,Description="OAC for your-blog-bucket",SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3 \
           --region us-east-1
         ```
       - Note the `OriginAccessControlId`.
     - Select `BlogOAC` in the origin settings.
   - **S3 Bucket Access**: Select **Yes, update the bucket policy** (adds):
     ```json
     {
         "Version": "2008-10-17",
         "Statement": [
             {
                 "Effect": "Allow",
                 "Principal": {
                     "Service": "cloudfront.amazonaws.com"
                 },
                 "Action": "s3:GetObject",
                 "Resource": "arn:aws:s3:::your-blog-bucket/*",
                 "Condition": {
                     "StringEquals": {
                         "AWS:SourceArn": "arn:aws:cloudfront::<your-account-id>:distribution/<cloudfront-id>"
                     }
                 }
             }
         ]
     }
     ```
3. **Default Cache Behavior**:
   - **Viewer Protocol Policy**: **Redirect HTTP to HTTPS**.
   - **Allowed HTTP Methods**: **GET, HEAD, OPTIONS**.
   - **Cache Policy**: **CachingOptimized**.
4. **Settings**:
   - **Default Root Object**: `index.html`.
   - **Price Class**: **Use all edge locations** or **Use only North America and Europe** (~$0.085/GB vs. ~$0.12/GB).
   - Click **Create Distribution** (~5–10 minutes).
5. **Configure SPA Routing**:
   - Select distribution → **Error Responses** → **Create Custom Error Response**.
   - For `403` and `404`:
     - **HTTP Error Code**: 403 and 404.
     - **Customize Error Response**: Yes.
     - **Response Page Path**: `/index.html`.
     - **HTTP Response Code**: `200 OK`.
   - Save changes.

## Step 7: Add Blog Assets
1. **Create Asset Bucket**:
   - Go to **S3** → **Create bucket** → Name: `your-blog-assets`.
   - **Region**: `us-west-1`.
   - **Object Ownership**: **Bucket owner enforced**.
   - **Block Public Access**: Enable.
2. **Upload Assets**:
   - Upload images/videos (e.g., `post1.jpg`) to `s3://your-blog-assets/images/` (~$0.005 for 1,000 files, free tier: 2,000 PUTs).
3. **Add to CloudFront**:
   - In **CloudFront** → Select distribution → **Origins** → **Create Origin**.
   - **Origin Domain**: `your-blog-assets.s3.amazonaws.com`.
   - **Origin Access**: Select `BlogOAC` or create a new OAC.
   - **Behavior**: Create a behavior for `/assets/*` to route to `your-blog-assets`.
4. **Integrate**:
   - Store URLs (e.g., `https://<cloudfront-id>.cloudfront.net/assets/post1.jpg`) in DynamoDB’s `imageUrl` field.
   - Update `BlogService` or templates to use these URLs.

## Step 8: Test the Application
1. **Invalidate CloudFront Cache**:
   - In **CloudFront** → Select distribution → **Invalidations** → **Create Invalidation** → Path: `/*`.
   - Cost: ~$0.005 (free tier: 1,000 invalidations).
2. **Test Locally**:
   - Run: `ng serve --open`.
   - Update `BlogService` `apiUrl` to `https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts`.
   - Verify:
     - Application compiles without errors.
     - Navigation links (`/`, `/blog`, `/admin`, `/post/1753978262517`) work.
     - Global styles (`#1a1a1a` background, `.container`) apply.
     - Admin page allows creating, editing, and deleting posts.
     - Post detail page loads without CORS errors.
   - Open DevTools (F12) → **Network** tab:
     - Filter for `https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts/1753978262517`.
     - Verify response includes `Access-Control-Allow-Origin: *`.
     - Check for `200` or `404` status.
3. **Test Deployed App**:
   - Access `https://<cloudfront-id>.cloudfront.net`.
   - Verify:
     - App loads with `index.html`.
     - Routes work (`/blog`, `/post/1753978262517`, `/admin`).
     - API calls succeed (`GET /posts`, `POST /posts`, `PUT /posts/{postId}`, `DELETE /posts/{postId}`).
     - Global and component styles render correctly.
   - Check DevTools (F12) → **Network**:
     - `styles.css` loads from `https://<cloudfront-id>.cloudfront.net/styles.css`.
     - API calls return `200` with `Access-Control-Allow-Origin: *`.
   - Test S3 access: `curl https://your-blog-bucket.s3.amazonaws.com/styles.css` (should fail due to OAC).
4. **CLI Tests**:
   ```bash
   curl -v -X GET https://<cloudfront-id>.cloudfront.net/styles.css
   curl -v -X GET https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts
   curl -v -X POST https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts -H "Content-Type: application/json" -d '{"title":"Test","content":"Content"}'
   curl -v -X GET https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts/1753978262517
   curl -v -X PUT https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts/1753978262517 -H "Content-Type: application/json" -d '{"title":"Updated","content":"Updated Content","createdAt":"2025-08-02T12:00:00Z"}'
   curl -v -X DELETE https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts/1753978262517
   ```

## Step 9: Add Custom Domain (Optional)
1. **Register Domain**:
   - In **Route 53**, register `yourdomain.com` (~$12/year).
2. **Create Hosted Zone**:
   - Create hosted zone for `yourdomain.com`.
   - Update nameservers if registered elsewhere.
3. **Request SSL Certificate**:
   - In **AWS Certificate Manager (ACM)** → Request certificate (in `us-east-1`).
   - **Domain Name**: `blog.yourdomain.com`.
   - **Validation**: DNS validation (add CNAME to Route 53).
4. **Update CloudFront**:
   - Select distribution → **General** → **Edit**.
   - **Alternate Domain Name**: `blog.yourdomain.com`.
   - **SSL Certificate**: Select ACM certificate.
   - Save (~5–10 minutes).
5. **Add Route 53 Record**:
   - In **Route 53** → Hosted Zone → **Create Record**.
   - **Record Name**: `blog.yourdomain.com`.
   - **Record Type**: `A`.
   - **Alias**: Enable, select CloudFront distribution.
   - Save.

## Step 10: Costs
- **CloudFront**: 1 GB × $0.085 × 12 = ~$1.02; 1,000 requests × $0.0075/10,000 × 12 = ~$0.009; invalidation ~$0.005 (free tier: 1 TB, 1,000 invalidations).
- **S3**: 10 MB × $0.023/GB × 12 = ~$0.00276; 50 files × $0.005/1,000 = ~$0.00025 (free tier: 5 GB, 2,000 PUTs).
- **Assets**: 1 GB × $0.023 × 12 = ~$0.276; 1,000 files × $0.005/1,000 = ~$0.005 (free tier).
- **API Gateway**: 1,000 requests × $3.50/million × 12 = ~$0.042 (free tier: 1M calls).
- **DynamoDB**: 1 MB × $1.25/GB × 12 = ~$0.015 (free tier: 25 GB).
- **Lambda**: 1,000 requests × $0.20/million = ~$0.0002 (free tier: 1M requests).
- **Route 53**: ~$0.50/month + $12/year = ~$18.
- **Total First Year**: ~$1.35 (free tier covers most services); with Route 53: ~$19.35.

## Step 11: Troubleshooting
- **TS2305 Error**:
  - Ensure `app.config.ts` does not import `provideForms`.
  - Verify `ReactiveFormsModule` is imported in `AdminComponent`.
  - Check Angular version: `ng --version` (should be 19.x).
- **Navigation Links Not Working**:
  - Ensure `RouterLink` is imported in `AppComponent`:
    ```typescript
    imports: [RouterOutlet, RouterLink, CommonModule]
    ```
  - Verify `app.config.ts` includes `provideRouter(routes)`.
  - Check `index.html` for `<base href="/">`.
  - Test locally: `ng serve --open` and use DevTools to check for errors.
- **CORS Missing Allow Origin**:
  - Verify CORS for `GET /posts/{postId}` in API Gateway:
    - Enable CORS for `GET` and `OPTIONS` methods.
    - Ensure `Access-Control-Allow-Origin: *` (or specific origin) in **Integration Response**.
    - Confirm `OPTIONS /posts/{postId}` has correct headers.
  - Check `getPost` Lambda function returns:
    ```python
    "headers": {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
        "Access-Control-Allow-Methods": "GET,PUT,DELETE,OPTIONS"
    }
    ```
  - Redeploy API: **Actions** → **Deploy API** → `prod`.
  - Test: `curl -v -X GET https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts/1753978262517`.
- **Access Denied on S3**:
  - Verify OAC policy and bucket policy.
  - Test: `curl https://your-blog-bucket.s3.amazonaws.com/styles.css` (should fail due to OAC).
- **Routing Issues**:
  - Ensure 403/404 redirects to `/index.html` (Step 6).
- **API Errors**:
  - Test: `curl -v -X PUT https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod/posts/1753978262517 -H "Content-Type: application/json" -d '{"title":"Updated","content":"Updated Content"}'`.
  - Verify CORS headers and `Authorization: NONE`, `API Key Required: false`.
- **Styles Not Applied**:
  - Check `dist/blog-app/styles.css` in S3.
  - Verify `angular.json` includes `styles.css`.
  - Invalidate CloudFront cache (`/*`, ~$0.005).
- **Update Issues**:
  - Log `editForm.value` in `onUpdate()` to verify payload.
  - Check Lambda logs in **CloudWatch** for `updatePost` errors.

## Step 12: Lessons Learned
- **Angular 19 Standalone**: Uses separate `.html` and `.css` files, with `styles.css` for dark theme. Forms are enabled via `ReactiveFormsModule` in components, not global providers.
- **Navigation**: Requires `RouterLink` in `AppComponent` for `routerLink` directives.
- **BlogService**: Includes `updatePost` for full CRUD.
- **CloudFront OAC**: Secures S3 with `BlogOAC`.
- **Lambda CORS**: Headers resolve CORS issues.
- **API Gateway**: `Authorization: NONE` avoids authentication errors.
- **Costs**: Minimal (~$1.35/year, free tier).
