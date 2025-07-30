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
         provideZoneChangeDetection({ eventCoalescing
