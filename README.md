# Tutorial: Building and Hosting a Standalone Angular 19 Blog Application on AWS with Custom Routing

This tutorial updates the previous guide to create an **Angular 19** standalone application with **Home**, **About**, **Profile**, and **Blog pages**, supporting dynamic blog post management (create, edit, delete) using **reactive forms** (no `NgModel`). It aligns with your existing `main.ts` and `app.config.ts` structure, using a separate `app.routes.ts` file for routing, and hosts the app on **AWS S3 + CloudFront** with a serverless backend (**API Gateway + Lambda + DynamoDB**) and optional **Route 53** for a custom domain. It minimizes **PUT charges** by zipping Angular source for archival storage in S3 Glacier/Deep Archive (not in the AWS free tier) and integrates with your school file storage (documents, videos, Angular projects).

## Prerequisites
- **Node.js** (v18+): Install from [nodejs.org](https://nodejs.org).
- **Angular CLI**: `npm install -g @angular/cli@19`.
- **AWS Account**: Sign up at [aws.amazon.com](https://aws.amazon.com) (free tier: 5 GB S3 Standard, 1 TB CloudFront egress, 25 GB DynamoDB, 1M Lambda requests).
- **AWS CLI** (optional): Install via [AWS CLI docs](https://aws.amazon.com/cli/).
- **Domain** (optional): For custom domain (e.g., `blog.yourdomain.com`).
- **Tools**: Git, code editor (e.g., VS Code), unzip tool (e.g., 7-Zip).

## Step 1: Create the Angular Application
1. **Initialize Project** (if not already done):
   ```bash
   ng new blog-app --standalone --routing --style=css
   ```
   - Confirms your standalone setup with routing and CSS.
2. **Generate Components and Service**:
   ```bash
   ng generate component home --standalone
   ng generate component about --standalone
   ng generate component profile --standalone
   ng generate component blog --standalone
   ng generate component admin --standalone
   ng generate service blog
   ```
   - `home`: Welcome page.
   - `about`: Blog information.
   - `profile`: Personal info (e.g., portfolio).
   - `blog`: Lists blog posts.
   - `admin`: Form for creating/editing/deleting posts.
   - `blog.service`: Handles API calls.
3. **Update Routing**:
   - Edit `app.routes.ts` (aligns with your setup):
     ```typescript
     import { Routes } from '@angular/router';
     import { HomeComponent } from './home/home.component';
     import { AboutComponent } from './about/about.component';
     import { ProfileComponent } from './profile/profile.component';
     import { BlogComponent } from './blog/blog.component';
     import { AdminComponent } from './admin/admin.component';

     export const routes: Routes = [
       { path: '', component: HomeComponent },
       { path: 'about', component: AboutComponent },
       { path: 'profile', component: ProfileComponent },
       { path: 'blog', component: BlogComponent },
       { path: 'admin', component: AdminComponent },
       { path: '**', redirectTo: '' }
     ];
     ```
   - Includes a wildcard route for 404 handling.
4. **Update App Config**:
   - Edit `app.config.ts` to include `provideHttpClient` and `provideReactiveForms`:
     ```typescript
     import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
     import { provideRouter } from '@angular/router';
     import { provideHttpClient } from '@angular/common/http';
     import { provideReactiveForms } from '@angular/forms';
     import { routes } from './app.routes';

     export const appConfig: ApplicationConfig = {
       providers: [
         provideZoneChangeDetection({ eventCoalescing: true }),
         provideRouter(routes),
         provideHttpClient(),
         provideReactiveForms()
       ]
     };
     ```
5. **Update Main**:
   - Ensure `main.ts` matches your setup:
     ```typescript
     import { bootstrapApplication } from '@angular/platform-browser';
     import { AppComponent } from './app/app.component';
     import { appConfig } from './app/app.config';

     bootstrapApplication(AppComponent, appConfig);
     ```
6. **Create Blog Service**:
   - Edit `blog.service.ts`:
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
       private apiUrl = 'https://<your-api-id>.execute-api.<region>.amazonaws.com/prod/posts';

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
   - Replace `<your-api-id>` and `<region>` with your API Gateway endpoint (set later).
7. **Implement Components**:
   - **Home** (`home.component.html`):
     ```html
     <h1>Welcome to My Blog</h1>
     <p>Explore my posts, about, and profile!</p>
     <a routerLink="/blog">View Blog</a>
     ```
     - Edit `home.component.ts`:
       ```typescript
       import { Component } from '@angular/core';
       import { RouterLink } from '@angular/router';

       @Component({
         selector: 'app-home',
         standalone: true,
         imports: [RouterLink],
         templateUrl: './home.component.html'
       })
       export class HomeComponent {}
       ```
   - **About** (`about.component.html`):
     ```html
     <h1>About</h1>
     <p>This blog shares my academic and coding journey.</p>
     ```
     - Edit `about.component.ts`:
       ```typescript
       import { Component } from '@angular/core';

       @Component({
         selector: 'app-about',
         standalone: true,
         templateUrl: './about.component.html'
       })
       export class AboutComponent {}
       ```
   - **Profile** (`profile.component.html`):
     ```html
     <h1>Profile</h1>
     <p>Name: [Your Name]</p>
     <p>Background: ICT Master's, Angular Developer</p>
     <p>Projects: <a routerLink="/blog">See my blog posts</a></p>
     ```
     - Edit `profile.component.ts`:
       ```typescript
       import { Component } from '@angular/core';
       import { RouterLink } from '@angular/router';

       @Component({
         selector: 'app-profile',
         standalone: true,
         imports: [RouterLink],
         templateUrl: './profile.component.html'
       })
       export class ProfileComponent {}
       ```
   - **Blog** (`blog.component.html`):
     ```html
     <h1>Blog Posts</h1>
     <div *ngFor="let post of posts">
       <h3>{{ post.title }}</h3>
       <p>{{ post.content }}</p>
       <img *ngIf="post.imageUrl" [src]="post.imageUrl" alt="Post Image">
     </div>
     <a routerLink="/admin">Manage Posts</a>
     ```
     - Edit `blog.component.ts`:
       ```typescript
       import { Component, OnInit } from '@angular/core';
       import { CommonModule } from '@angular/common';
       import { RouterLink } from '@angular/router';
       import { BlogService, BlogPost } from '../blog.service';

       @Component({
         selector: 'app-blog',
         standalone: true,
         imports: [CommonModule, RouterLink],
         templateUrl: './blog.component.html'
       })
       export class BlogComponent implements OnInit {
         posts: BlogPost[] = [];

         constructor(private blogService: BlogService) {}

         ngOnInit() {
           this.blogService.getPosts().subscribe(posts => this.posts = posts);
         }
       }
       ```
   - **Admin** (reactive forms, `admin.component.html`):
     ```html
     <h1>Manage Posts</h1>
     <form [formGroup]="postForm" (ngSubmit)="savePost()">
       <input formControlName="postId" placeholder="Post ID" readonly>
       <input formControlName="title" placeholder="Title" required>
       <textarea formControlName="content" placeholder="Content" required></textarea>
       <input formControlName="imageUrl" placeholder="Image URL">
       <button type="submit" [disabled]="postForm.invalid">Save</button>
       <button type="button" (click)="resetForm()">Clear</button>
     </form>
     <div *ngFor="let post of posts">
       <h3>{{ post.title }}</h3>
       <button (click)="editPost(post)">Edit</button>
       <button (click)="deletePost(post.postId)">Delete</button>
     </div>
     ```
     - Edit `admin.component.ts`:
       ```typescript
       import { Component, OnInit } from '@angular/core';
       import { CommonModule } from '@angular/common';
       import { ReactiveFormsModule, FormBuilder, FormGroup, Validators } from '@angular/forms';
       import { BlogService, BlogPost } from '../blog.service';

       @Component({
         selector: 'app-admin',
         standalone: true,
         imports: [CommonModule, ReactiveFormsModule],
         templateUrl: './admin.component.html'
       })
       export class AdminComponent implements OnInit {
         posts: BlogPost[] = [];
         postForm: FormGroup;

         constructor(private blogService: BlogService, private fb: FormBuilder) {
           this.postForm = this.fb.group({
             postId: [''],
             title: ['', Validators.required],
             content: ['', Validators.required],
             imageUrl: ['']
           });
         }

         ngOnInit() {
           this.blogService.getPosts().subscribe(posts => this.posts = posts);
         }

         savePost() {
           const post: BlogPost = this.postForm.value;
           if (post.postId) {
             this.blogService.updatePost(post.postId, post).subscribe(() => this.resetForm());
           } else {
             post.postId = Date.now().toString();
             this.blogService.createPost(post).subscribe(() => this.resetForm());
           }
         }

         editPost(post: BlogPost) {
           this.postForm.setValue({
             postId: post.postId,
             title: post.title,
             content: post.content,
             imageUrl: post.imageUrl || ''
           });
         }

         deletePost(postId: string) {
           this.blogService.deletePost(postId).subscribe(() => {
             this.posts = this.posts.filter(p => p.postId !== postId);
           });
         }

         resetForm() {
           this.postForm.reset({ postId: '', title: '', content: '', imageUrl: '' });
           this.blogService.getPosts().subscribe(posts => this.posts = posts);
         }
       }
       ```
8. **Update App Component**:
   - Edit `app.component.html`:
     ```html
     <nav>
       <a routerLink="/">Home</a>
       <a routerLink="/about">About</a>
       <a routerLink="/profile">Profile</a>
       <a routerLink="/blog">Blog</a>
       <a routerLink="/admin">Admin</a>
     </nav>
     <router-outlet></router-outlet>
     ```
   - Edit `app.component.ts`:
     ```typescript
     import { Component } from '@angular/core';
     import { RouterOutlet, RouterLink } from '@angular/router';

     @Component({
       selector: 'app-root',
       standalone: true,
       imports: [RouterOutlet, RouterLink],
       templateUrl: './app.component.html'
     })
     export class AppComponent {}
     ```
9. **Test Locally**:
   ```bash
   ng serve
   ```
   - Verify at `http://localhost:4200` (navigate Home, About, Profile, Blog, Admin; test reactive form in Admin).

## Step 2: Set Up AWS Backend
1. **Create DynamoDB Table**:
   - In AWS Console, go to DynamoDB → Create Table.
   - Name: `BlogPosts`.
   - Partition Key: `postId` (String).
   - Attributes: `title` (String), `content` (String), `imageUrl` (String), `createdAt` (String).
   - Pricing: On-demand (~$1.25/TB/month, free tier: 25 GB).
2. **Create Lambda Functions**:
   - Go to Lambda → Create Function (Python 3.9).
   - **Create Post**:
     ```python
     import json
     import boto3
     from datetime import datetime

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             body = json.loads(event["body"])
             post_id = body["postId"]
             title = body["title"]
             content = body["content"]
             image_url = body.get("imageUrl", "")
             
             table.put_item(
                 Item={
                     "postId": post_id,
                     "title": title,
                     "content": content,
                     "imageUrl": image_url,
                     "createdAt": datetime.utcnow().isoformat()
                 }
             )
             return {
                 "statusCode": 200,
                 "body": json.dumps({"message": "Post created", "postId": post_id})
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **Read Posts**:
     ```python
     import json
     import boto3

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             response = table.scan()
             return {
                 "statusCode": 200,
                 "body": json.dumps(response["Items"])
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **Update Post**:
     ```python
     import json
     import boto3

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             body = json.loads(event["body"])
             post_id = event["pathParameters"]["postId"]
             table.update_item(
                 Key={"postId": post_id},
                 UpdateExpression="SET title = :t, content = :c, imageUrl = :i",
                 ExpressionAttributeValues={
                     ":t": body["title"],
                     ":c": body["content"],
                     ":i": body.get("imageUrl", "")
                 }
             )
             return {
                 "statusCode": 200,
                 "body": json.dumps({"message": "Post updated"})
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **Delete Post**:
     ```python
     import json
     import boto3

     dynamodb = boto3.resource("dynamodb")
     table = dynamodb.Table("BlogPosts")

     def lambda_handler(event, context):
         try:
             post_id = event["pathParameters"]["postId"]
             table.delete_item(Key={"postId": post_id})
             return {
                 "statusCode": 200,
                 "body": json.dumps({"message": "Post deleted"})
             }
         except Exception as e:
             return {
                 "statusCode": 500,
                 "body": json.dumps({"error": str(e)})
             }
     ```
   - **IAM Role**: Attach a role with `dynamodb:PutItem`, `dynamodb:GetItem`, `dynamodb:Scan`, `dynamodb:UpdateItem`, `dynamodb:DeleteItem` permissions.
3. **Set Up API Gateway**:
   - Go to API Gateway → Create API → REST API.
   - Create resources: `/posts`, `/posts/{postId}`.
   - Add methods:
     - `POST /posts` → Lambda `create_post`.
     - `GET /posts` → Lambda `read_posts`.
     - `PUT /posts/{postId}` → Lambda `update_post`.
     - `DELETE /posts/{postId}` → Lambda `delete_post`.
   - Enable CORS for all methods.
   - Deploy to a stage (e.g., `prod`).
   - Update `blog.service.ts` with the endpoint (e.g., `https://<api-id>.execute-api.us-east-1.amazonaws.com/prod`).

## Step 3: Host on S3 + CloudFront
1. **Build Angular App**:
   ```bash
   ng build --configuration production
   ```
   - Outputs `dist/blog-app/` (~10–50 files).
2. **Create S3 Bucket**:
   - In AWS Console, create `your-blog-bucket`.
   - Enable **Static website hosting** (Index: `index.html`).
   - Upload `dist/blog-app/` to S3 Standard (~$0.00025 PUT charges for 50 files, free tier: 2,000 PUTs).
   - Set bucket policy for public read or OAC with CloudFront:
     ```json
     {
         "Version": "2012-10-17",
         "Statement": [
             {
                 "Effect": "Allow",
                 "Principal": "*",
                 "Action": "s3:GetObject",
                 "Resource": "arn:aws:s3:::your-blog-bucket/*"
             }
         ]
     }
     ```
3. **Create CloudFront Distribution**:
   - Go to CloudFront → Create Distribution.
   - Origin: `your-blog-bucket`.
   - Origin Access: Enable **OAC**, update bucket policy via Console.
   - Viewer Protocol: Redirect HTTP to HTTPS.
   - Default Root Object: `index.html`.
   - Error Responses: Redirect 403/404 to `/index.html` (HTTP 200) for SPA routing.
   - Deploy (~5–10 minutes).
4. **Test**: Access at `https://<cloudfront-id>.cloudfront.net`.

## Step 4: Add Custom Domain (Optional)
1. **Register Domain**:
   - In Route 53, register `yourdomain.com` (~$12/year).
2. **Create Hosted Zone**:
   - Create a hosted zone for `yourdomain.com`.
   - Update nameservers if registered elsewhere.
3. **SSL Certificate**:
   - In AWS Certificate Manager, create a certificate for `blog.yourdomain.com` (free).
   - Attach to CloudFront.
4. **Add Record**:
   - In Route 53 hosted zone, create an A record for `blog.yourdomain.com`, alias to CloudFront distribution.

## Step 5: Manage Blog Assets
1. **Create Asset Bucket**:
   - Create `your-blog-assets` (S3 Standard for instant access).
2. **Upload Assets**:
   - Upload images/videos via AWS Console (~$0.005 for 1,000 files, free tier: 2,000 PUTs).
   - Get CloudFront URLs (e.g., `https://<cloudfront-id>.cloudfront.net/assets/post1.jpg`).
3. **Integrate**:
   - Store URLs in DynamoDB’s `imageUrl` field.
   - Add to CloudFront as a second origin (`/assets/*`).

## Step 6: Handle Zipped Angular Source
- **Archive**: Zip `blog-app` as `blog-app.zip` to minimize PUT charges (~$0.00001 for S3-IA).
- **Upload**: Store in `school-files-archive-<yourname>` (S3-IA → Glacier → Deep Archive, not in free tier).
- **Lifecycle Policy**:
  - In AWS Console, go to S3 → `school-files-archive-<yourname>` → Management → Create Lifecycle Rule.
  - Transition to Glacier (30 days, ~$4/TB/month), Deep Archive (90 days, ~$1/TB/month).
- **Unzip for Edits**:
  - **Local**: Download (restore from Glacier/Deep Archive, ~$0.01–$0.02/GB, ~$0.09/GB egress, free tier: 100 GB egress), unzip with 7-Zip, edit, rebuild, re-upload `dist/` (~$0.00025 for 50 files).
  - **Lambda**: Use the unzip Lambda from prior responses (~$0.01 + ~$0.0005 for 50 files).
- **Cost**: ~$0.0012/year for 100 MB in Deep Archive (not free tier).

## Step 7: Test and Update
- **Test**: Access `https://blog.yourdomain.com` or CloudFront URL. Verify navigation (Home, About, Profile, Blog, Admin) and CRUD in Admin (reactive forms).s
- **Update**:
  - Edit Angular code, rebuild (`ng build`), upload `dist/` to S3 (~$0.00025).
  - Invalidate CloudFront cache (`/*`, ~$0.005 after 1,000 free/month, free tier: 1,000 invalidations).

## Costs
- **Frontend (S3+CloudFront)**:
  - PUT: 50 files × $0.005/1,000 = ~$0.00025 (free tier: 2,000 PUTs).
  - Storage: 10 MB × $0.023/GB × 12 = ~$0.00276 (free tier: 5 GB).
  - CloudFront: 1 GB × $0.085 × 12 = ~$1.02 + 1,000 × $0.0075/10,000 × 12 = ~$0.009 (free tier: 1 TB).
- **Assets**: 1 GB × $0.023 × 12 = ~$0.276; 1,000 files × $0.005/1,000 = ~$0.005 (free tier).
- **Backend**:
  - DynamoDB: 1 MB × $1.25/GB × 12 = ~$0.015 (free tier: 25 GB).
  - Lambda: 1,000 requests × $0.20/million = ~$0.0002 (free tier: 1M requests).
  - API Gateway: 1,000 requests × $3.50/million × 12 = ~$0.042 (free tier: 1M calls).
- **Zipped Source**: 100 MB × $0.00099/GB × 12 = ~$0.0012 (not free tier).
- **Route 53**: ~$0.50/month + $12/year = ~$18.
- **Total First Year**:
  - ~$1.35 (free tier covers S3 Standard, CloudFront, DynamoDB, Lambda, API Gateway).
  - With Route 53: ~$19.35.
- **School Files**: 1 TB in Deep Archive (not free tier) = ~$31/year (S3-IA → Glacier → Deep Archive, ~$0.05 PUT for 5,010 files with zipped projects).

## Integration with School Files
- **Storage**: Store zipped Angular projects and school files in `school-files-archive-<yourname>` (S3-IA → Glacier → Deep Archive, ~$31/year for 1 TB, not free tier).
- **Access**: Restore Glacier/Deep Archive files (~$0.01–$0.02/GB, free tier: 100 GB egress) for edits.
- **Sharing**: Use presigned URLs (~$0.09/GB) or CloudFront (~$0.085/GB, free tier) for videos/projects.
- **Lifecycle**: No transitions from Glacier to Standard (as asked); restore manually for access.

## Lessons Learned
- **Angular 19**: Standalone components and reactive forms streamline development, avoiding `NgModel`.
- **AWS**: S3+CloudFront is cost-effective for static hosting; serverless backend minimizes costs.
- **PUT Charges**: Zipping Angular source saves ~$0.01 per 1,000 files.
- **Glacier/Deep Archive**: Not in free tier, use for archival (~$1–$4/TB/month), restore for hosting.
