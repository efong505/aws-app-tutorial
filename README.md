# Tutorial: Building an Angular 19 Blog and Hosting with CloudFront, S3, API Gateway, Lambda, and DynamoDB

This tutorial guides you through creating an **Angular 19 standalone blog application**, hosting its static files (`dist/blog-app/`) on **Amazon S3** with **CloudFront** as a CDN, and integrating dynamic CRUD operations via **AWS API Gateway**, **Lambda**, and **DynamoDB**. It addresses the issue of not seeing the **Create Control Setting** option for **Origin Access Control (OAC)** in CloudFront, ensuring secure S3 access. The setup supports your requirements: `main.ts`, `app.config.ts`, `app.routes.ts`, styled components with dark theme, reactive forms, S3-hosted assets, and school file storage (zipped Angular projects in S3-IA → Glacier → Deep Archive, not in free tier). It resolves prior issues (CORS for `DELETE /posts/{postId}`, authentication for `GET /posts/{postId}`) and minimizes costs (~$1.35/year for blog, free tier; ~$31/year for 1 TB school files).

## Prerequisites
- **AWS Account**: Sign up at [aws.amazon.com](https://aws.amazon.com) (free tier: 1 TB CloudFront egress, 1,000 invalidations, 5 GB S3 Standard, 2,000 PUT requests, 1M API Gateway calls, 25 GB DynamoDB, 1M Lambda requests).
- **IAM Permissions**: Ensure your IAM user/role has:
  ```json
  {
      "Effect": "Allow",
      "Action": [
          "s3:*",
          "cloudfront:*",
          "dynamodb:*",
          "lambda:*",
          "execute-api:*",
          "iam:PassRole",
          "iam:GetRole",
          "iam:CreateRole",
          "iam:PutRolePolicy"
      ],
      "Resource": "*"
  }
  ```
- **Node.js and Angular CLI**: Node.js 20.x, Angular CLI 19.x (`npm install -g @angular/cli@19`).
- **AWS CLI**: Installed and configured (`aws configure`).
- **Tools**: Code editor (e.g., VS Code), 7-Zip for zipping.

## Step 1: Create Angular 19 Standalone Blog Application
1. **Create Project**:
   ```bash
   ng new blog-app --standalone --style=css --routing
   cd blog-app
   ```
   - Select **CSS**, enable **routing**, no SSR.
2. **Install Dependencies**:
   ```bash
   npm install bootstrap @ng-bootstrap/ng-bootstrap rxjs
   ```
   - Adds Bootstrap for styling and reactive forms support.
3. **Set Up Project Structure**:
   ```bash
   ng generate component home
   ng generate component blog
   ng generate component post
   ng generate component admin
   ng generate service blog
   ```
4. **Configure Bootstrap**:
   - In `src/styles.css`:
     ```css
     @import "~bootstrap/dist/css/bootstrap.min.css";
     body {
       background-color: #1a1a1a; /* Dark theme */
       color: #ffffff;
     }
     .navbar {
       position: fixed;
       top: 0;
       width: 100%;
       z-index: 1000;
     }
     .card {
       background-color: #2c2c2c;
       border: 1px solid #444;
     }
     .container {
       margin-top: 70px; /* Offset for fixed navbar */
     }
     ```
   - In `angular.json`, add Bootstrap JS:
     ```json
     "scripts": [
       "node_modules/bootstrap/dist/js/bootstrap.bundle.min.js"
     ]
     ```
5. **Configure App**:
   - **src/main.ts**:
     ```typescript
     import { bootstrapApplication } from '@angular/platform-browser';
     import { provideRouter } from '@angular/router';
     import { provideHttpClient } from '@angular/common/http';
     import { provideAnimations } from '@angular/platform-browser/animations';
     import { AppComponent } from './app/app.component';
     import { appRoutes } from './app/app.routes';
     import { importProvidersFrom } from '@angular/core';
     import { ReactiveFormsModule } from '@angular/forms';
     import { NgbModule } from '@ng-bootstrap/ng-bootstrap';

     bootstrapApplication(AppComponent, {
       providers: [
         provideRouter(appRoutes),
         provideHttpClient(),
         provideAnimations(),
         importProvidersFrom(ReactiveFormsModule, NgbModule)
       ]
     }).catch(err => console.error(err));
     ```
   - **src/app/app.component.ts**:
     ```typescript
     import { Component } from '@angular/core';
     import { RouterOutlet } from '@angular/router';
     import { CommonModule } from '@angular/common';
     import { NgbNavModule } from '@ng-bootstrap/ng-bootstrap';

     @Component({
       selector: 'app-root',
       standalone: true,
       imports: [RouterOutlet, CommonModule, NgbNavModule],
       template: `
         <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
           <div class="container">
             <a class="navbar-brand" href="/">Blog</a>
             <ul class="navbar-nav">
               <li class="nav-item"><a class="nav-link" routerLink="/">Home</a></li>
               <li class="nav-item"><a class="nav-link" routerLink="/blog">Blog</a></li>
               <li class="nav-item"><a class="nav-link" routerLink="/admin">Admin</a></li>
             </ul>
           </div>
         </nav>
         <div class="container">
           <router-outlet></router-outlet>
         </div>
       `
     })
     export class AppComponent {}
     ```
   - **src/app/app.routes.ts**:
     ```typescript
     import { Routes } from '@angular/router';
     import { HomeComponent } from './home/home.component';
     import { BlogComponent } from './blog/blog.component';
     import { PostComponent } from './post/post.component';
     import { AdminComponent } from './admin/admin.component';

     export const appRoutes: Routes = [
       { path: '', component: HomeComponent },
       { path: 'blog', component: BlogComponent },
       { path: 'post/:id', component: PostComponent },
       { path: 'admin', component: AdminComponent },
       { path: '**', redirectTo: '' }
     ];
     ```
6. **Create Blog Service**:
   - **src/app/blog.service.ts**:
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
7. **Create Components**:
   - **HomeComponent** (`src/app/home/home.component.ts`):
     ```typescript
     import { Component } from '@angular/core';

     @Component({
       selector: 'app-home',
       standalone: true,
       template: `
         <div class="container">
           <h1>Welcome to the Blog</h1>
           <p>Explore our posts or manage content in the admin panel.</p>
         </div>
       `
     })
     export class HomeComponent {}
     ```
   - **BlogComponent** (`src/app/blog/blog.component.ts`):
     ```typescript
     import { Component, OnInit } from '@angular/core';
     import { CommonModule } from '@angular/common';
     import { RouterLink } from '@angular/router';
     import { BlogService, BlogPost } from '../blog.service';

     @Component({
       selector: 'app-blog',
       standalone: true,
       imports: [CommonModule, RouterLink],
       template: `
         <div class="container">
           <h2>Blog Posts</h2>
           <div class="row">
             <div class="col-md-4" *ngFor="let post of posts">
               <div class="card mb-4">
                 <img *ngIf="post.imageUrl" [src]="post.imageUrl" class="card-img-top" alt="{{post.title}}">
                 <div class="card-body">
                   <h5 class="card-title">{{post.title}}</h5>
                   <p class="card-text">{{post.content | slice:0:100}}...</p>
                   <a [routerLink]="['/post', post.postId]" class="btn btn-primary">Read More</a>
                 </div>
               </div>
             </div>
           </div>
         </div>
       `
     })
     export class BlogComponent implements OnInit {
       posts: BlogPost[] = [];
       constructor(private blogService: BlogService) {}

       ngOnInit() {
         this.blogService.getPosts().subscribe(posts => this.posts = posts);
       }
     }
     ```
   - **PostComponent** (`src/app/post/post.component.ts`):
     ```typescript
     import { Component, OnInit } from '@angular/core';
     import { CommonModule } from '@angular/common';
     import { ActivatedRoute } from '@angular/router';
     import { BlogService, BlogPost } from '../blog.service';

     @Component({
       selector: 'app-post',
       standalone: true,
       imports: [CommonModule],
       template: `
         <div class="container" *ngIf="post">
           <h2>{{post.title}}</h2>
           <img *ngIf="post.imageUrl" [src]="post.imageUrl" class="img-fluid mb-3" alt="{{post.title}}">
           <p>{{post.content}}</p>
           <p><small>Posted on: {{post.createdAt | date:'medium'}}</small></p>
         </div>
       `
     })
     export class PostComponent implements OnInit {
       post: BlogPost | undefined;
       constructor(private route: ActivatedRoute, private blogService: BlogService) {}

       ngOnInit() {
         const id = this.route.snapshot.paramMap.get('id')!;
         this.blogService.getPost(id).subscribe(post => this.post = post);
       }
     }
     ```
   - **AdminComponent** (`src/app/admin/admin.component.ts`):
     ```typescript
     import { Component, OnInit } from '@angular/core';
     import { CommonModule } from '@angular/common';
     import { ReactiveFormsModule, FormBuilder, FormGroup, Validators } from '@angular/forms';
     import { BlogService, BlogPost } from '../blog.service';

     @Component({
       selector: 'app-admin',
       standalone: true,
       imports: [CommonModule, ReactiveFormsModule],
       template: `
         <div class="container">
           <h2>Manage Posts</h2>
           <form [formGroup]="postForm" (ngSubmit)="onSubmit()">
             <div class="mb-3">
               <label class="form-label">Title</label>
               <input type="text" class="form-control" formControlName="title">
             </div>
             <div class="mb-3">
               <label class="form-label">Content</label>
               <textarea class="form-control" formControlName="content"></textarea>
             </div>
             <div class="mb-3">
               <label class="form-label">Image URL</label>
               <input type="text" class="form-control" formControlName="imageUrl">
             </div>
             <button type="submit" class="btn btn-primary" [disabled]="postForm.invalid">Create Post</button>
           </form>
           <h3 class="mt-5">Existing Posts</h3>
           <ul class="list-group">
             <li class="list-group-item d-flex justify-content-between" *ngFor="let post of posts">
               {{post.title}}
               <div>
                 <button class="btn btn-danger btn-sm" (click)="deletePost(post.postId)">Delete</button>
               </div>
             </li>
           </ul>
         </div>
       `
     })
     export class AdminComponent implements OnInit {
       postForm: FormGroup;
       posts: BlogPost[] = [];

       constructor(private fb: FormBuilder, private blogService: BlogService) {
         this.postForm = this.fb.group({
           title: ['', Validators.required],
           content: ['', Validators.required],
           imageUrl: [''],
           createdAt: [new Date().toISOString()]
         });
       }

       ngOnInit() {
         this.blogService.getPosts().subscribe(posts => this.posts = posts);
       }

       onSubmit() {
         const post: BlogPost = {
           postId: Date.now().toString(),
           ...this.postForm.value
         };
         this.blogService.createPost(post).subscribe(() => {
           this.posts.push(post);
           this.postForm.reset({ createdAt: new Date().toISOString() });
         });
       }

       deletePost(postId: string) {
         this.blogService.deletePost(postId).subscribe(() => {
           this.posts = this.posts.filter(p => p.postId !== postId);
         });
       }
     }
     ```
8. **Test Locally**:
   - Run `ng serve`.
   - Access `http://localhost:4200`.
   - Verify navigation (`/`, `/blog`, `/post/:id`, `/admin`), styling (dark theme, fixed nav, responsive grid), and form functionality.

## Step 2: Set Up AWS Backend
1. **Create DynamoDB Table**:
   - In **AWS Console** → **DynamoDB** → **Tables** → **Create table**.
   - **Table name**: `BlogPosts`.
   - **Partition key**: `postId` (String).
   - **Billing mode**: Provisioned (1 RCU/WCU, free tier: 25 GB).
2. **Create Lambda Functions**:
   - In **Lambda** → **Create function** for each: `createPost`, `readPosts`, `getPost`, `deletePost`.
   - **Runtime**: Python 3.12.
   - **IAM Role**:
     ```json
     {
         "Version": "2012-10-17",
         "Statement": [
             {
                 "Effect": "Allow",
                 "Action": ["dynamodb:PutItem", "dynamodb:Scan", "dynamodb:GetItem", "dynamodb:DeleteItem"],
                 "Resource": "arn:aws:dynamodb:us-west-1:371751795928:table/BlogPosts"
             },
             {
                 "Effect": "Allow",
                 "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
                 "Resource": "*"
             }
         ]
     }
     ```
   - **Code**:
     - `createPost`:
       ```python
       import json
       import boto3

       dynamodb = boto3.resource("dynamodb")
       table = dynamodb.Table("BlogPosts")

       def lambda_handler(event, context):
           try:
               body = json.loads(event.get("body", "{}"))
               post = {
                   "postId": body.get("postId"),
                   "title": body.get("title"),
                   "content": body.get("content"),
                   "imageUrl": body.get("imageUrl", ""),
                   "createdAt": body.get("createdAt")
               }
               table.put_item(Item=post)
               return {
                   "statusCode": 201,
                   "headers": {
                       "Access-Control-Allow-Origin": "*",
                       "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                       "Access-Control-Allow-Methods": "POST,GET,OPTIONS"
                   },
                   "body": json.dumps({"message": "Post created"})
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
     - `readPosts`:
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
                       "Access-Control-Allow-Methods": "GET,OPTIONS"
                   },
                   "body": json.dumps(posts)
               }
           except Exception as e:
               return {
                   "statusCode": 500,
                   "headers": {
                       "Access-Control-Allow-Origin": "*",
                       "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                       "Access-Control-Allow-Methods": "GET,OPTIONS"
                   },
                   "body": json.dumps({"error": str(e)})
               }
       ```
     - `getPost`:
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
                           "Access-Control-Allow-Methods": "GET,DELETE,OPTIONS"
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
                           "Access-Control-Allow-Methods": "GET,DELETE,OPTIONS"
                       },
                       "body": json.dumps({"error": f"Post with postId {post_id} not found"})
                   }
               return {
                   "statusCode": 200,
                   "headers": {
                       "Access-Control-Allow-Origin": "*",
                       "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                       "Access-Control-Allow-Methods": "GET,DELETE,OPTIONS"
                   },
                   "body": json.dumps(post)
               }
           except Exception as e:
               return {
                   "statusCode": 500,
                   "headers": {
                       "Access-Control-Allow-Origin": "*",
                       "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                       "Access-Control-Allow-Methods": "GET,DELETE,OPTIONS"
                   },
                   "body": json.dumps({"error": str(e)})
               }
       ```
     - `deletePost`:
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
                           "Access-Control-Allow-Methods": "DELETE,GET,OPTIONS"
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
                           "Access-Control-Allow-Methods": "DELETE,GET,OPTIONS"
                       },
                       "body": json.dumps({"error": f"Post with postId {post_id} not found"})
                   }
               return {
                   "statusCode": 200,
                   "headers": {
                       "Access-Control-Allow-Origin": "*",
                       "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                       "Access-Control-Allow-Methods": "DELETE,GET,OPTIONS"
                   },
                   "body": json.dumps({"message": f"Post {post_id} deleted"})
               }
           except Exception as e:
               return {
                   "statusCode": 500,
                   "headers": {
                       "Access-Control-Allow-Origin": "*",
                       "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token",
                       "Access-Control-Allow-Methods": "DELETE,GET,OPTIONS"
                   },
                   "body": json.dumps({"error": str(e)})
               }
       ```
3. **Configure API Gateway**:
   - In **API Gateway** → **Create API** → **REST API** → Name: `BlogAPI`.
   - **Resources**:
     - Create `/posts`:
       - **POST**: Integration with `createPost` Lambda (Proxy integration).
       - **GET**: Integration with `readPosts` Lambda (Proxy integration).
       - **OPTIONS**: Mock integration, set headers:
         ```
         Access-Control-Allow-Origin: *
         Access-Control-Allow-Headers: Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token
         Access-Control-Allow-Methods: POST,GET,OPTIONS
         ```
     - Create `/posts/{postId}`:
       - **GET**: Integration with `getPost` Lambda (Proxy integration).
       - **DELETE**: Integration with `deletePost` Lambda (Proxy integration).
       - **OPTIONS**: Mock integration, set headers:
         ```
         Access-Control-Allow-Origin: *
         Access-Control-Allow-Headers: Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token
         Access-Control-Allow-Methods: GET,DELETE,OPTIONS
         ```
   - **Method Request** (for all methods):
     - **Authorization**: `NONE`.
     - **API Key Required**: `false` (resolves "Missing Authentication Token").
   - **Enable CORS**:
     - Select `/posts` → **Actions** → **Enable CORS** → Check `POST`, `GET`, `OPTIONS`.
     - Select `/posts/{postId}` → **Enable CORS** → Check `GET`, `DELETE`, `OPTIONS`.
   - **Deploy API**:
     - **Actions** → **Deploy API** → Stage: `prod`.
     - Note endpoint: `https://8xspcdvz22.execute-api.us-west-1.amazonaws.com/prod`.

## Step 3: Build and Upload Angular App to S3
1. **Build App**:
   ```bash
   ng build --configuration production
   ```
   - Outputs `dist/blog-app/` (~10–50 files, ~10 MB):
     ```
     dist/blog-app/
     ├── index.html
     ├── main.js
     ├── polyfills.js
     ├── styles.css
     ├── assets/
     │   ├── images/
     │   ├── favicon.ico
     │   └── ...
     └── ...
     ```
2. **Create S3 Bucket**:
   - In **S3** → **Create bucket** → Name: `your-blog-bucket` (us-west-1).
   - **Object Ownership**: **Bucket owner enforced**.
   - **Block Public Access**: Enable all.
3. **Upload Files**:
   - Go to `your-blog-bucket` → **Upload**.
   - Select **contents** of `dist/blog-app/` (e.g., `index.html`, `main.js`, `assets/`).
   - Upload (~$0.00025 for 50 files, free tier: 2,000 PUT requests).
   - Verify:
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
     └── ...
     ```
   - Use AWS CLI (optional):
     ```bash
     aws s3 sync dist/blog-app/ s3://your-blog-bucket/ --delete --region us-west-1
     ```

## Step 4: Create CloudFront Distribution with OAC
1. **Create Distribution**:
   - In **AWS Console** → **CloudFront** → **Create Distribution**.
   - **Origin Domain**: `your-blog-bucket.s3.amazonaws.com` (REST API endpoint).
   - **Origin Access**:
     - Select **Origin access control settings (recommended)**.
     - If **Create Control Setting** is missing, go to **Security** → **Origin access** → **Create origin access control**.
     - **Name**: `BlogOAC`.
     - **Origin Type**: S3.
     - **Signing Behavior**: Sign requests (recommended).
     - **Signing Protocol**: SigV4.
     - Click **Create**.
   - **S3 Bucket Access**: **Yes, update the bucket policy**.
   - **Viewer Protocol Policy**: **Redirect HTTP to HTTPS**.
   - **Allowed HTTP Methods**: **GET, HEAD, OPTIONS**.
   - **Cache Policy**: **CachingOptimized**.
   - **Default Root Object**: `index.html`.
   - Click **Create Distribution** (~5–10 minutes).
2. **If “Create Control Setting” Missing**:
   - Navigate to **CloudFront** → **Security** → **Origin access**.
   - If not visible, use AWS CLI:
     ```bash
     aws cloudfront create-origin-access-control \
       --origin-access-control-config \
         Name=BlogOAC,Description="OAC for your-blog-bucket",SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3 \
       --region us-east-1
     ```
   - Note `OriginAccessControlId`.
   - Update distribution:
     - **CloudFront** → Select distribution → **Origins** → Edit `your-blog-bucket.s3.amazonaws.com`.
     - Select `BlogOAC` under **Origin access control settings**.
     - **S3 bucket access**: **Yes, update the bucket policy**.
3. **Verify Bucket Policy**:
   - In **S3** → `your-blog-bucket` → **Permissions** → **Bucket policy**:
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
                         "AWS:SourceArn": "arn:aws:cloudfront::371751795928:distribution/<cloudfront-id>"
                     }
                 }
             }
         ]
     }
     ```
4. **Set Error Responses**:
   - In **CloudFront** → Select distribution → **Error Responses**.
   - For `403` and `404`:
     - **Customize Error Response**: Yes.
     - **Response Page Path**: `/index.html`.
     - **HTTP Response Code**: `200 OK`.
5. **Invalidate Cache**:
   - **Invalidations** → **Create Invalidation** → `/*` (~$0.005, free tier: 1,000 invalidations).

## Step 5: Test the Application
1. **Access CloudFront**:
   - Open `https://<cloudfront-id>.cloudfront.net`.
   - Verify:
     - App loads (`index.html`).
     - Routes work (`/`, `/blog`, `/post/1753938533864`, `/admin`).
     - API calls succeed (`GET /posts`, `GET /posts/{postId}`, `DELETE /posts/{postId}`).
     - Styling (dark theme, fixed nav, responsive grid) and forms work.
2. **DevTools**:
   - Open DevTools (F12) → **Network**.
   - Confirm static files load from `https://<cloudfront-id>.cloudfront.net`.
   - API calls return `200` with `Access-Control-Allow-Origin: *`.
3. **Test S3 Access**:
   - `curl https://your-blog-bucket.s3.amazonaws.com/index.html` (should fail).
   - `curl https://<cloudfront-id>.cloudfront.net/index.html` (should work).
4. **Troubleshooting**:
   - **404**: Ensure `index.html` is at bucket root.
   - **Routing Issues**: Verify 403/404 redirects to `/index.html`.
   - **API Errors**: Check CORS and authentication (`Authorization: NONE`, `API Key Required: false`).
   - **OAC Missing**:
     - Navigate to **CloudFront** → **Security** → **Origin access**.
     - Use CLI if needed (Step 4).

## Step 6: Add Custom Domain (Optional)
1. **Register Domain**:
   - **Route 53** → Register `yourdomain.com` (~$12/year).
2. **Hosted Zone**:
   - Create hosted zone for `yourdomain.com`.
3. **SSL Certificate**:
   - **ACM** (us-east-1) → Request certificate for `blog.yourdomain.com`.
   - Validate via DNS (add CNAME to Route 53).
4. **Update CloudFront**:
   - **Distributions** → Select distribution → **General** → **Edit**.
   - **Alternate Domain Name**: `blog.yourdomain.com`.
   - **SSL Certificate**: Select ACM certificate.
5. **Route 53 Record**:
   - **Hosted Zone** → **Create Record** → `blog.yourdomain.com` → **A** → Alias to CloudFront distribution.
6. **Test**: `https://blog.yourdomain.com`.

## Step 7: Manage Blog Assets
1. **Create Asset Bucket**:
   - Create `your-blog-assets` (S3 Standard).
   - Upload images/videos (~$0.005 for 1,000 files, free tier: 2,000 PUT requests).
2. **Add to CloudFront**:
   - **Origins** → Create origin for `your-blog-assets.s3.amazonaws.com`.
   - Use OAC (`BlogOAC` or new).
   - **Behavior**: Route `/assets/*` to `your-blog-assets`.
3. **Integrate**:
   - Store URLs (e.g., `https://<cloudfront-id>.cloudfront.net/assets/post1.jpg`) in DynamoDB `imageUrl`.
4. **Invalidate Cache**: `/assets/*` (~$0.005, free tier).

## Step 8: School File Storage
- **Archive**: Zip `blog-app` as `blog-app.zip` (~$0.00001 for S3-IA).
- **Upload**: To `school-files-archive-<yourname>` (S3-IA → Glacier → Deep Archive).
- **Lifecycle Policy**:
  - **S3** → `school-files-archive-<yourname>` → **Management** → **Create Lifecycle Rule**.
  - Transition: S3-IA (0 days, ~$12.50/TB/month) → Glacier (30 days, ~$4/TB/month) → Deep Archive (90 days, ~$1/TB/month).
- **Unzip for Edits**:
  - **Local**: Restore (~$0.01–$0.02/GB, free tier: 100 GB egress), unzip, edit, rebuild, re-upload (~$0.00025 for 50 files).
  - **Lambda**: Use unzip Lambda (~$0.01 + ~$0.0005).
- **Cost**: ~$31/year for 1 TB in Deep Archive (not free tier).

## Step 9: Costs
- **CloudFront**: ~$1.02/year (1 GB), ~$0.009/year (1,000 requests), ~$0.005/invalidation (free tier: 1 TB, 1,000 invalidations).
- **S3**: ~$0.00025 (50 files), ~$0.00276/year (10 MB), ~$0.276/year (1 GB assets).
- **API Gateway**: ~$0.042/year (1,000 requests, free tier: 1M).
- **DynamoDB**: ~$0.015/year (1 MB, free tier: 25 GB).
- **Lambda**: ~$0.0002/year (1,000 requests, free tier: 1M).
- **Zipped Source**: ~$0.0012/year (100 MB, Deep Archive, not free tier).
- **Route 53**: ~$18/year.
- **Total**: ~$1.35/year (free tier), ~$19.35 with Route 53, ~$31/year for 1 TB school files.

## Step 10: Lessons Learned
- **Angular 19**: Standalone components and reactive forms simplify development.
- **CloudFront**: OAC secures S3; 403/404 redirects enable SPA routing.
- **S3**: Upload `dist/blog-app/` contents to bucket root.
- **OAC**: Create via **Security** → **Origin access** or CLI if Console fails.
- **Costs**: Minimal with free tier; Deep Archive for school files (~$31/year).
- **AWS**: Scalable for dynamic blogs and large datasets.

## Step 11: Troubleshooting
- **OAC Not Visible**:
  - Check **CloudFront** → **Security** → **Origin access** or **Distributions** → **Origins** → **Edit**.
  - Use CLI: `aws cloudfront create-origin-access-control ...`.
  - Verify IAM permissions (`cloudfront:CreateOriginAccessControl`).
- **404 Errors**: Ensure `index.html` is at `s3://your-blog-bucket/index.html`.
- **Routing Issues**: Verify 403/404 redirects to `/index.html`.
- **API Errors**: Check CORS and authentication (`Authorization: NONE`, `API Key Required: false`).
- **Stale Content**: Invalidate cache (`/*`, ~$0.005, free tier).
