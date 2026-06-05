# Ionic Angular - Mobile-First UI Framework

## The Idea

**In plain English:** Ionic Angular is a toolkit that lets you write one set of code and turn it into a real app that works on iPhones, Android phones, and websites — it gives you ready-made buttons, menus, and screens that automatically look and feel like a native phone app.

**Real-world analogy:** Imagine a food truck that can park at a beach, a city street, or a shopping mall — the same truck, the same menu, but it adapts its setup to wherever it is. Then map each part explicitly:
- The food truck = your Ionic Angular app (one codebase)
- The beach, city street, and shopping mall = iOS, Android, and the web browser
- The adjustable awning and signage = Ionic's automatic platform styling that looks right on each platform
- The pre-built menu items = Ionic's ready-made UI components (buttons, lists, modals, etc.)

---

## Overview

Ionic Angular is a complete mobile-first UI framework for building hybrid mobile applications, Progressive Web Apps (PWAs), and desktop applications using Angular. It provides a library of mobile-optimized UI components, native device APIs through Capacitor, platform-specific styling, gestures, animations, and navigation patterns that feel native on iOS, Android, and web platforms.

## Installation

### Prerequisites

```bash
# Required dependencies
npm install @angular/core @angular/common @angular/forms @angular/router
```

### Installation Steps

```bash
# Install Ionic Angular
npm install @ionic/angular

# Install Ionic CLI globally (optional but recommended)
npm install -g @ionic/cli

# Create new Ionic Angular project
ionic start myApp blank --type=angular

# Or add to existing Angular project
npm install @ionic/angular @ionic/angular-toolkit
```

### Setup

**angular.json:**
```json
{
  "projects": {
    "app": {
      "architect": {
        "build": {
          "options": {
            "styles": [
              "src/theme/variables.scss",
              "src/global.scss"
            ],
            "assets": [
              {
                "glob": "**/*",
                "input": "node_modules/@ionic/angular/css",
                "output": "./css"
              }
            ]
          }
        }
      }
    }
  }
}
```

**app.module.ts:**
```typescript
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { RouteReuseStrategy } from '@angular/router';
import { IonicModule, IonicRouteStrategy } from '@ionic/angular';

import { AppComponent } from './app.component';
import { AppRoutingModule } from './app-routing.module';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    IonicModule.forRoot(),
    AppRoutingModule
  ],
  providers: [
    { provide: RouteReuseStrategy, useClass: IonicRouteStrategy }
  ],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

**src/global.scss:**
```scss
@import "~@ionic/angular/css/core.css";
@import "~@ionic/angular/css/normalize.css";
@import "~@ionic/angular/css/structure.css";
@import "~@ionic/angular/css/typography.css";
@import "~@ionic/angular/css/display.css";
@import "~@ionic/angular/css/padding.css";
@import "~@ionic/angular/css/float-elements.css";
@import "~@ionic/angular/css/text-alignment.css";
@import "~@ionic/angular/css/text-transformation.css";
@import "~@ionic/angular/css/flex-utils.css";
```

## Core Concepts

### 1. Platform-Specific Styling

Ionic automatically adapts component styling based on the platform (iOS/Android/Web).

### 2. Navigation with IonRouterOutlet

Ionic uses a specialized router outlet for mobile-style navigation with animations and gestures.

### 3. Gesture System

Built-in support for mobile gestures like swipe, pan, and pinch.

### 4. Native Device APIs

Integration with Capacitor for accessing native device features.

## Component Examples

### 1. Layout and Navigation

**App Shell with Navigation:**
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <ion-app>
      <ion-split-pane contentId="main-content">
        <!-- Side Menu -->
        <ion-menu contentId="main-content" type="overlay">
          <ion-header>
            <ion-toolbar color="primary">
              <ion-title>Menu</ion-title>
            </ion-toolbar>
          </ion-header>
          <ion-content>
            <ion-list>
              <ion-menu-toggle auto-hide="false" *ngFor="let page of appPages">
                <ion-item 
                  [routerLink]="page.url" 
                  routerDirection="root"
                  [routerLinkActive]="'active-item'">
                  <ion-icon [name]="page.icon" slot="start"></ion-icon>
                  <ion-label>{{page.title}}</ion-label>
                </ion-item>
              </ion-menu-toggle>
            </ion-list>

            <ion-list>
              <ion-list-header>Account</ion-list-header>
              <ion-item button (click)="onLogout()">
                <ion-icon name="log-out" slot="start"></ion-icon>
                <ion-label>Logout</ion-label>
              </ion-item>
            </ion-list>
          </ion-content>
        </ion-menu>

        <!-- Main Content -->
        <div id="main-content">
          <ion-router-outlet></ion-router-outlet>
        </div>
      </ion-split-pane>
    </ion-app>
  `,
  styles: [`
    .active-item {
      --background: var(--ion-color-primary-tint);
      --color: var(--ion-color-primary-contrast);
    }
  `]
})
export class AppComponent {
  appPages = [
    { title: 'Home', url: '/home', icon: 'home' },
    { title: 'Profile', url: '/profile', icon: 'person' },
    { title: 'Messages', url: '/messages', icon: 'mail' },
    { title: 'Settings', url: '/settings', icon: 'settings' }
  ];

  onLogout() {
    console.log('Logout');
  }
}
```

**Page with Header and Footer:**
```typescript
import { Component } from '@angular/core';
import { MenuController } from '@ionic/angular';

@Component({
  selector: 'app-home',
  template: `
    <ion-header [translucent]="true">
      <ion-toolbar color="primary">
        <ion-buttons slot="start">
          <ion-menu-button></ion-menu-button>
        </ion-buttons>
        <ion-title>Home</ion-title>
        <ion-buttons slot="end">
          <ion-button (click)="onSearch()">
            <ion-icon name="search" slot="icon-only"></ion-icon>
          </ion-button>
          <ion-button (click)="onNotifications()">
            <ion-icon name="notifications" slot="icon-only"></ion-icon>
            <ion-badge color="danger" *ngIf="notificationCount > 0">
              {{notificationCount}}
            </ion-badge>
          </ion-button>
        </ion-buttons>
      </ion-toolbar>

      <!-- Searchbar (optional) -->
      <ion-toolbar *ngIf="showSearch">
        <ion-searchbar 
          [(ngModel)]="searchTerm"
          (ionChange)="onSearchChange($event)"
          placeholder="Search...">
        </ion-searchbar>
      </ion-toolbar>

      <!-- Segment (Tab-like navigation) -->
      <ion-toolbar>
        <ion-segment 
          [(ngModel)]="selectedSegment" 
          (ionChange)="onSegmentChange($event)">
          <ion-segment-button value="all">
            <ion-label>All</ion-label>
          </ion-segment-button>
          <ion-segment-button value="favorites">
            <ion-label>Favorites</ion-label>
          </ion-segment-button>
          <ion-segment-button value="recent">
            <ion-label>Recent</ion-label>
          </ion-segment-button>
        </ion-segment>
      </ion-toolbar>
    </ion-header>

    <ion-content [fullscreen]="true">
      <!-- Refresher -->
      <ion-refresher slot="fixed" (ionRefresh)="doRefresh($event)">
        <ion-refresher-content
          pullingIcon="chevron-down-circle-outline"
          pullingText="Pull to refresh"
          refreshingSpinner="circles"
          refreshingText="Refreshing...">
        </ion-refresher-content>
      </ion-refresher>

      <!-- Content based on segment -->
      <div [ngSwitch]="selectedSegment">
        <div *ngSwitchCase="'all'">
          <ion-list>
            <ion-item *ngFor="let item of items" button (click)="onItemClick(item)">
              <ion-avatar slot="start">
                <img [src]="item.avatar" />
              </ion-avatar>
              <ion-label>
                <h2>{{item.title}}</h2>
                <p>{{item.description}}</p>
              </ion-label>
              <ion-note slot="end">{{item.time}}</ion-note>
            </ion-item>
          </ion-list>
        </div>

        <div *ngSwitchCase="'favorites'">
          <ion-list>
            <ion-item *ngFor="let item of favoriteItems">
              <ion-icon name="star" color="warning" slot="start"></ion-icon>
              <ion-label>{{item.title}}</ion-label>
            </ion-item>
          </ion-list>
        </div>

        <div *ngSwitchCase="'recent'">
          <ion-list>
            <ion-item *ngFor="let item of recentItems">
              <ion-icon name="time" slot="start"></ion-icon>
              <ion-label>{{item.title}}</ion-label>
              <ion-note slot="end">{{item.timestamp}}</ion-note>
            </ion-item>
          </ion-list>
        </div>
      </div>

      <!-- Infinite Scroll -->
      <ion-infinite-scroll 
        threshold="100px" 
        (ionInfinite)="loadMore($event)">
        <ion-infinite-scroll-content
          loadingSpinner="bubbles"
          loadingText="Loading more...">
        </ion-infinite-scroll-content>
      </ion-infinite-scroll>
    </ion-content>

    <ion-footer>
      <ion-toolbar>
        <ion-buttons slot="start">
          <ion-button (click)="onFilter()">
            <ion-icon name="filter" slot="start"></ion-icon>
            Filter
          </ion-button>
        </ion-buttons>
        <ion-buttons slot="end">
          <ion-button (click)="onSort()">
            <ion-icon name="swap-vertical" slot="start"></ion-icon>
            Sort
          </ion-button>
        </ion-buttons>
      </ion-toolbar>
    </ion-footer>
  `
})
export class HomePage {
  showSearch = false;
  searchTerm = '';
  selectedSegment = 'all';
  notificationCount = 3;

  items = [
    {
      title: 'Item 1',
      description: 'Description for item 1',
      avatar: 'assets/avatar1.jpg',
      time: '2m ago'
    },
    {
      title: 'Item 2',
      description: 'Description for item 2',
      avatar: 'assets/avatar2.jpg',
      time: '15m ago'
    }
  ];

  favoriteItems = [
    { title: 'Favorite 1' },
    { title: 'Favorite 2' }
  ];

  recentItems = [
    { title: 'Recent 1', timestamp: '1h ago' },
    { title: 'Recent 2', timestamp: '2h ago' }
  ];

  constructor(private menuCtrl: MenuController) {}

  onSearch() {
    this.showSearch = !this.showSearch;
  }

  onNotifications() {
    console.log('Show notifications');
  }

  onSearchChange(event: any) {
    console.log('Search:', event.detail.value);
  }

  onSegmentChange(event: any) {
    console.log('Segment:', event.detail.value);
  }

  doRefresh(event: any) {
    setTimeout(() => {
      console.log('Refreshed');
      event.target.complete();
    }, 2000);
  }

  loadMore(event: any) {
    setTimeout(() => {
      console.log('Load more');
      event.target.complete();
    }, 1000);
  }

  onItemClick(item: any) {
    console.log('Item clicked:', item);
  }

  onFilter() {
    console.log('Filter');
  }

  onSort() {
    console.log('Sort');
  }
}
```

### 2. Forms and Inputs

**Complete Form Example:**
```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';
import { AlertController, LoadingController } from '@ionic/angular';

@Component({
  selector: 'app-user-form',
  template: `
    <ion-header>
      <ion-toolbar color="primary">
        <ion-buttons slot="start">
          <ion-back-button></ion-back-button>
        </ion-buttons>
        <ion-title>User Profile</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
        <!-- Text Input -->
        <ion-item>
          <ion-label position="floating">Full Name</ion-label>
          <ion-input 
            formControlName="fullName" 
            type="text"
            placeholder="Enter your name">
          </ion-input>
        </ion-item>
        <ion-text color="danger" *ngIf="isFieldInvalid('fullName')">
          <p class="error-text">Name is required (min 3 characters)</p>
        </ion-text>

        <!-- Email Input -->
        <ion-item>
          <ion-label position="floating">Email</ion-label>
          <ion-input 
            formControlName="email" 
            type="email"
            placeholder="email@example.com">
          </ion-input>
          <ion-icon name="mail-outline" slot="end"></ion-icon>
        </ion-item>
        <ion-text color="danger" *ngIf="isFieldInvalid('email')">
          <p class="error-text">Valid email is required</p>
        </ion-text>

        <!-- Password Input -->
        <ion-item>
          <ion-label position="floating">Password</ion-label>
          <ion-input 
            formControlName="password" 
            [type]="showPassword ? 'text' : 'password'"
            placeholder="Enter password">
          </ion-input>
          <ion-button 
            fill="clear" 
            slot="end"
            (click)="togglePassword()">
            <ion-icon 
              [name]="showPassword ? 'eye-off-outline' : 'eye-outline'"
              slot="icon-only">
            </ion-icon>
          </ion-button>
        </ion-item>

        <!-- Phone Input -->
        <ion-item>
          <ion-label position="floating">Phone Number</ion-label>
          <ion-input 
            formControlName="phone" 
            type="tel"
            placeholder="+1 (555) 000-0000">
          </ion-input>
        </ion-item>

        <!-- Select -->
        <ion-item button (click)="selectCountry()">
          <ion-label>Country</ion-label>
          <ion-note slot="end">
            {{userForm.get('country')?.value || 'Select'}}
          </ion-note>
        </ion-item>

        <!-- Date Picker -->
        <ion-item button (click)="selectDate()">
          <ion-label>Birth Date</ion-label>
          <ion-note slot="end">
            {{userForm.get('birthDate')?.value || 'Select date'}}
          </ion-note>
        </ion-item>

        <!-- Textarea -->
        <ion-item>
          <ion-label position="floating">Bio</ion-label>
          <ion-textarea 
            formControlName="bio" 
            rows="4"
            placeholder="Tell us about yourself">
          </ion-textarea>
        </ion-item>
        <ion-note>
          {{userForm.get('bio')?.value?.length || 0}} / 500 characters
        </ion-note>

        <!-- Toggle -->
        <ion-item>
          <ion-label>Email Notifications</ion-label>
          <ion-toggle 
            formControlName="emailNotifications" 
            slot="end">
          </ion-toggle>
        </ion-item>

        <!-- Checkbox -->
        <ion-item>
          <ion-label>SMS Notifications</ion-label>
          <ion-checkbox 
            formControlName="smsNotifications" 
            slot="end">
          </ion-checkbox>
        </ion-item>

        <!-- Radio Group -->
        <ion-list-header>
          <ion-label>Account Type</ion-label>
        </ion-list-header>
        <ion-radio-group formControlName="accountType">
          <ion-item>
            <ion-label>Personal</ion-label>
            <ion-radio slot="start" value="personal"></ion-radio>
          </ion-item>
          <ion-item>
            <ion-label>Business</ion-label>
            <ion-radio slot="start" value="business"></ion-radio>
          </ion-item>
        </ion-radio-group>

        <!-- Range Slider -->
        <ion-item>
          <ion-label>Age: {{userForm.get('age')?.value}}</ion-label>
        </ion-item>
        <ion-item>
          <ion-range 
            formControlName="age"
            [min]="18"
            [max]="100"
            [pin]="true"
            [snaps]="true"
            color="primary">
            <ion-label slot="start">18</ion-label>
            <ion-label slot="end">100</ion-label>
          </ion-range>
        </ion-item>

        <!-- Chips (Tags) -->
        <ion-item>
          <ion-label position="stacked">Interests</ion-label>
          <div class="chips-container">
            <ion-chip *ngFor="let interest of interests" (click)="removeInterest(interest)">
              <ion-label>{{interest}}</ion-label>
              <ion-icon name="close-circle"></ion-icon>
            </ion-chip>
            <ion-button fill="clear" (click)="addInterest()">
              <ion-icon name="add-circle" slot="start"></ion-icon>
              Add
            </ion-button>
          </div>
        </ion-item>

        <!-- Buttons -->
        <div class="button-container">
          <ion-button 
            type="submit" 
            expand="block"
            [disabled]="!userForm.valid">
            <ion-icon name="checkmark" slot="start"></ion-icon>
            Save Profile
          </ion-button>

          <ion-button 
            type="button" 
            expand="block"
            fill="outline"
            (click)="onReset()">
            Reset Form
          </ion-button>
        </div>
      </form>
    </ion-content>
  `,
  styles: [`
    .error-text {
      padding: 0 16px;
      font-size: 12px;
      margin-top: 4px;
    }

    .chips-container {
      display: flex;
      flex-wrap: wrap;
      gap: 8px;
      padding: 8px 0;
    }

    .button-container {
      padding: 16px;
    }

    ion-button {
      margin-bottom: 8px;
    }
  `]
})
export class UserFormPage implements OnInit {
  userForm!: FormGroup;
  showPassword = false;
  interests: string[] = ['Technology', 'Sports'];

  countries = ['USA', 'UK', 'Canada', 'Australia'];

  constructor(
    private fb: FormBuilder,
    private alertCtrl: AlertController,
    private loadingCtrl: LoadingController
  ) {}

  ngOnInit() {
    this.userForm = this.fb.group({
      fullName: ['', [Validators.required, Validators.minLength(3)]],
      email: ['', [Validators.required, Validators.email]],
      password: ['', [Validators.required, Validators.minLength(8)]],
      phone: [''],
      country: [''],
      birthDate: [''],
      bio: ['', Validators.maxLength(500)],
      emailNotifications: [true],
      smsNotifications: [false],
      accountType: ['personal'],
      age: [25]
    });
  }

  isFieldInvalid(fieldName: string): boolean {
    const field = this.userForm.get(fieldName);
    return !!(field && field.invalid && field.touched);
  }

  togglePassword() {
    this.showPassword = !this.showPassword;
  }

  async selectCountry() {
    const alert = await this.alertCtrl.create({
      header: 'Select Country',
      inputs: this.countries.map(country => ({
        type: 'radio',
        label: country,
        value: country,
        checked: this.userForm.get('country')?.value === country
      })),
      buttons: [
        {
          text: 'Cancel',
          role: 'cancel'
        },
        {
          text: 'OK',
          handler: (value) => {
            this.userForm.patchValue({ country: value });
          }
        }
      ]
    });

    await alert.present();
  }

  async selectDate() {
    const alert = await this.alertCtrl.create({
      header: 'Select Date',
      inputs: [
        {
          type: 'date',
          name: 'birthDate',
          value: this.userForm.get('birthDate')?.value
        }
      ],
      buttons: [
        {
          text: 'Cancel',
          role: 'cancel'
        },
        {
          text: 'OK',
          handler: (data) => {
            this.userForm.patchValue({ birthDate: data.birthDate });
          }
        }
      ]
    });

    await alert.present();
  }

  async addInterest() {
    const alert = await this.alertCtrl.create({
      header: 'Add Interest',
      inputs: [
        {
          type: 'text',
          name: 'interest',
          placeholder: 'Enter interest'
        }
      ],
      buttons: [
        {
          text: 'Cancel',
          role: 'cancel'
        },
        {
          text: 'Add',
          handler: (data) => {
            if (data.interest && !this.interests.includes(data.interest)) {
              this.interests.push(data.interest);
            }
          }
        }
      ]
    });

    await alert.present();
  }

  removeInterest(interest: string) {
    this.interests = this.interests.filter(i => i !== interest);
  }

  async onSubmit() {
    if (this.userForm.valid) {
      const loading = await this.loadingCtrl.create({
        message: 'Saving...',
        duration: 2000
      });

      await loading.present();

      console.log('Form submitted:', this.userForm.value);

      await loading.dismiss();

      const alert = await this.alertCtrl.create({
        header: 'Success',
        message: 'Profile saved successfully!',
        buttons: ['OK']
      });

      await alert.present();
    }
  }

  onReset() {
    this.userForm.reset({
      emailNotifications: true,
      smsNotifications: false,
      accountType: 'personal',
      age: 25
    });
    this.interests = [];
  }
}
```

### 3. Cards and Lists

**Social Feed Example:**
```typescript
import { Component } from '@angular/core';
import { ActionSheetController } from '@ionic/angular';

@Component({
  selector: 'app-feed',
  template: `
    <ion-header>
      <ion-toolbar color="primary">
        <ion-title>Feed</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <!-- Story/Status Bar -->
      <ion-slides [options]="slideOpts" class="story-slides">
        <ion-slide *ngFor="let story of stories">
          <div class="story-item">
            <ion-avatar>
              <img [src]="story.avatar" />
            </ion-avatar>
            <p>{{story.name}}</p>
          </div>
        </ion-slide>
      </ion-slides>

      <!-- Feed Cards -->
      <ion-card *ngFor="let post of posts" class="feed-card">
        <!-- Card Header -->
        <ion-card-header>
          <div class="card-header-content">
            <ion-avatar>
              <img [src]="post.author.avatar" />
            </ion-avatar>
            <div class="author-info">
              <ion-card-title>{{post.author.name}}</ion-card-title>
              <ion-card-subtitle>{{post.timestamp}}</ion-card-subtitle>
            </div>
            <ion-button fill="clear" (click)="showPostActions(post)">
              <ion-icon name="ellipsis-horizontal" slot="icon-only"></ion-icon>
            </ion-button>
          </div>
        </ion-card-header>

        <!-- Card Content -->
        <ion-card-content>
          <p>{{post.content}}</p>
          
          <img *ngIf="post.image" [src]="post.image" class="post-image" />

          <!-- Reactions -->
          <div class="reactions">
            <ion-chip color="primary">
              <ion-icon name="heart"></ion-icon>
              <ion-label>{{post.likes}}</ion-label>
            </ion-chip>
            <ion-chip>
              <ion-icon name="chatbubble"></ion-icon>
              <ion-label>{{post.comments}} comments</ion-label>
            </ion-chip>
            <ion-chip>
              <ion-icon name="share"></ion-icon>
              <ion-label>{{post.shares}} shares</ion-label>
            </ion-chip>
          </div>
        </ion-card-content>

        <!-- Action Buttons -->
        <ion-row class="action-buttons">
          <ion-col>
            <ion-button 
              fill="clear" 
              size="small"
              (click)="onLike(post)"
              [color]="post.liked ? 'danger' : 'medium'">
              <ion-icon 
                [name]="post.liked ? 'heart' : 'heart-outline'" 
                slot="start">
              </ion-icon>
              Like
            </ion-button>
          </ion-col>
          <ion-col>
            <ion-button fill="clear" size="small" (click)="onComment(post)">
              <ion-icon name="chatbubble-outline" slot="start"></ion-icon>
              Comment
            </ion-button>
          </ion-col>
          <ion-col>
            <ion-button fill="clear" size="small" (click)="onShare(post)">
              <ion-icon name="share-outline" slot="start"></ion-icon>
              Share
            </ion-button>
          </ion-col>
        </ion-row>
      </ion-card>
    </ion-content>

    <!-- Floating Action Button -->
    <ion-fab vertical="bottom" horizontal="end" slot="fixed">
      <ion-fab-button color="primary" (click)="onCreatePost()">
        <ion-icon name="add"></ion-icon>
      </ion-fab-button>
    </ion-fab>
  `,
  styles: [`
    .story-slides {
      height: 100px;
      padding: 8px 0;
    }

    .story-item {
      text-align: center;
    }

    .story-item ion-avatar {
      width: 60px;
      height: 60px;
      border: 2px solid var(--ion-color-primary);
    }

    .story-item p {
      font-size: 12px;
      margin-top: 4px;
    }

    .feed-card {
      margin: 16px;
    }

    .card-header-content {
      display: flex;
      align-items: center;
      gap: 12px;
    }

    .card-header-content ion-avatar {
      width: 40px;
      height: 40px;
    }

    .author-info {
      flex: 1;
    }

    .author-info ion-card-title {
      font-size: 16px;
      font-weight: 600;
    }

    .author-info ion-card-subtitle {
      font-size: 12px;
    }

    .post-image {
      width: 100%;
      border-radius: 8px;
      margin-top: 8px;
    }

    .reactions {
      display: flex;
      gap: 8px;
      margin-top: 12px;
    }

    .reactions ion-chip {
      font-size: 12px;
    }

    .action-buttons {
      border-top: 1px solid var(--ion-color-light);
      padding: 4px 0;
    }

    .action-buttons ion-button {
      width: 100%;
    }
  `]
})
export class FeedPage {
  slideOpts = {
    slidesPerView: 5,
    spaceBetween: 10
  };

  stories = [
    { name: 'Your Story', avatar: 'assets/you.jpg' },
    { name: 'John', avatar: 'assets/john.jpg' },
    { name: 'Jane', avatar: 'assets/jane.jpg' },
    { name: 'Bob', avatar: 'assets/bob.jpg' }
  ];

  posts = [
    {
      id: 1,
      author: { name: 'John Doe', avatar: 'assets/john.jpg' },
      timestamp: '2 hours ago',
      content: 'Just finished an amazing hike! The views were incredible.',
      image: 'assets/hike.jpg',
      likes: 45,
      comments: 12,
      shares: 3,
      liked: false
    },
    {
      id: 2,
      author: { name: 'Jane Smith', avatar: 'assets/jane.jpg' },
      timestamp: '5 hours ago',
      content: 'Loving this new coffee shop downtown. Great atmosphere!',
      image: null,
      likes: 28,
      comments: 7,
      shares: 1,
      liked: true
    }
  ];

  constructor(private actionSheetCtrl: ActionSheetController) {}

  async showPostActions(post: any) {
    const actionSheet = await this.actionSheetCtrl.create({
      header: 'Post Actions',
      buttons: [
        {
          text: 'Save Post',
          icon: 'bookmark-outline',
          handler: () => {
            console.log('Save clicked');
          }
        },
        {
          text: 'Report Post',
          icon: 'flag-outline',
          role: 'destructive',
          handler: () => {
            console.log('Report clicked');
          }
        },
        {
          text: 'Cancel',
          icon: 'close',
          role: 'cancel'
        }
      ]
    });

    await actionSheet.present();
  }

  onLike(post: any) {
    post.liked = !post.liked;
    post.likes += post.liked ? 1 : -1;
  }

  onComment(post: any) {
    console.log('Comment on post:', post.id);
  }

  onShare(post: any) {
    console.log('Share post:', post.id);
  }

  onCreatePost() {
    console.log('Create new post');
  }
}
```

### 4. Modals and Popovers

**Modal Service Usage:**
```typescript
import { Component } from '@angular/core';
import { ModalController, PopoverController } from '@ionic/angular';

@Component({
  selector: 'app-modal-demo',
  template: `
    <ion-header>
      <ion-toolbar>
        <ion-title>Modals & Popovers</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <ion-button expand="block" (click)="openModal()">
        Open Modal
      </ion-button>

      <ion-button expand="block" (click)="openFullModal()">
        Open Full Screen Modal
      </ion-button>

      <ion-button expand="block" (click)="openPopover($event)">
        Open Popover
      </ion-button>
    </ion-content>
  `
})
export class ModalDemoPage {
  constructor(
    private modalCtrl: ModalController,
    private popoverCtrl: PopoverController
  ) {}

  async openModal() {
    const modal = await this.modalCtrl.create({
      component: UserModalComponent,
      componentProps: {
        user: { name: 'John Doe', email: 'john@example.com' }
      },
      breakpoints: [0, 0.5, 0.75, 1],
      initialBreakpoint: 0.75
    });

    await modal.present();

    const { data } = await modal.onWillDismiss();
    console.log('Modal dismissed with:', data);
  }

  async openFullModal() {
    const modal = await this.modalCtrl.create({
      component: FullScreenModalComponent,
      presentingElement: await this.modalCtrl.getTop()
    });

    await modal.present();
  }

  async openPopover(event: any) {
    const popover = await this.popoverCtrl.create({
      component: PopoverMenuComponent,
      event: event,
      translucent: true
    });

    await popover.present();

    const { data } = await popover.onDidDismiss();
    console.log('Popover dismissed with:', data);
  }
}

// Modal Component
@Component({
  selector: 'app-user-modal',
  template: `
    <ion-header>
      <ion-toolbar>
        <ion-title>User Details</ion-title>
        <ion-buttons slot="end">
          <ion-button (click)="dismiss()">Close</ion-button>
        </ion-buttons>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <ion-list>
        <ion-item>
          <ion-label>
            <h2>Name</h2>
            <p>{{user.name}}</p>
          </ion-label>
        </ion-item>
        <ion-item>
          <ion-label>
            <h2>Email</h2>
            <p>{{user.email}}</p>
          </ion-label>
        </ion-item>
      </ion-list>

      <ion-button expand="block" (click)="save()">
        Save Changes
      </ion-button>
    </ion-content>
  `
})
export class UserModalComponent {
  user: any;

  constructor(private modalCtrl: ModalController) {}

  dismiss() {
    this.modalCtrl.dismiss();
  }

  save() {
    this.modalCtrl.dismiss({ saved: true });
  }
}

// Full Screen Modal
@Component({
  selector: 'app-full-screen-modal',
  template: `
    <ion-header>
      <ion-toolbar color="primary">
        <ion-buttons slot="start">
          <ion-button (click)="dismiss()">
            <ion-icon name="close" slot="icon-only"></ion-icon>
          </ion-button>
        </ion-buttons>
        <ion-title>Full Screen</ion-title>
        <ion-buttons slot="end">
          <ion-button (click)="save()">Save</ion-button>
        </ion-buttons>
      </ion-toolbar>
    </ion-header>

    <ion-content>
      <div class="ion-padding">
        <h1>Full Screen Modal</h1>
        <p>This modal takes up the entire screen.</p>
      </div>
    </ion-content>
  `
})
export class FullScreenModalComponent {
  constructor(private modalCtrl: ModalController) {}

  dismiss() {
    this.modalCtrl.dismiss();
  }

  save() {
    this.modalCtrl.dismiss({ saved: true });
  }
}

// Popover Component
@Component({
  selector: 'app-popover-menu',
  template: `
    <ion-list>
      <ion-item button (click)="select('edit')">
        <ion-icon name="create-outline" slot="start"></ion-icon>
        <ion-label>Edit</ion-label>
      </ion-item>
      <ion-item button (click)="select('share')">
        <ion-icon name="share-outline" slot="start"></ion-icon>
        <ion-label>Share</ion-label>
      </ion-item>
      <ion-item button (click)="select('delete')">
        <ion-icon name="trash-outline" slot="start" color="danger"></ion-icon>
        <ion-label color="danger">Delete</ion-label>
      </ion-item>
    </ion-list>
  `
})
export class PopoverMenuComponent {
  constructor(private popoverCtrl: PopoverController) {}

  select(action: string) {
    this.popoverCtrl.dismiss({ action });
  }
}
```

### 5. Native Device Features with Capacitor

**Camera and Geolocation:**
```typescript
import { Component } from '@angular/core';
import { Camera, CameraResultType, CameraSource } from '@capacitor/camera';
import { Geolocation } from '@capacitor/geolocation';
import { Toast } from '@capacitor/toast';
import { Share } from '@capacitor/share';

@Component({
  selector: 'app-native-features',
  template: `
    <ion-header>
      <ion-toolbar>
        <ion-title>Native Features</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <!-- Camera -->
      <ion-card>
        <ion-card-header>
          <ion-card-title>Camera</ion-card-title>
        </ion-card-header>
        <ion-card-content>
          <img *ngIf="photo" [src]="photo" class="photo-preview" />
          <ion-button expand="block" (click)="takePicture()">
            <ion-icon name="camera" slot="start"></ion-icon>
            Take Photo
          </ion-button>
          <ion-button expand="block" fill="outline" (click)="selectFromGallery()">
            <ion-icon name="images" slot="start"></ion-icon>
            Choose from Gallery
          </ion-button>
        </ion-card-content>
      </ion-card>

      <!-- Geolocation -->
      <ion-card>
        <ion-card-header>
          <ion-card-title>Location</ion-card-title>
        </ion-card-header>
        <ion-card-content>
          <ion-list *ngIf="location">
            <ion-item>
              <ion-label>
                <h3>Latitude</h3>
                <p>{{location.latitude}}</p>
              </ion-label>
            </ion-item>
            <ion-item>
              <ion-label>
                <h3>Longitude</h3>
                <p>{{location.longitude}}</p>
              </ion-label>
            </ion-item>
          </ion-list>
          <ion-button expand="block" (click)="getCurrentLocation()">
            <ion-icon name="location" slot="start"></ion-icon>
            Get Current Location
          </ion-button>
        </ion-card-content>
      </ion-card>

      <!-- Share -->
      <ion-card>
        <ion-card-header>
          <ion-card-title>Share</ion-card-title>
        </ion-card-header>
        <ion-card-content>
          <ion-button expand="block" (click)="shareContent()">
            <ion-icon name="share-social" slot="start"></ion-icon>
            Share Content
          </ion-button>
        </ion-card-content>
      </ion-card>

      <!-- Toast -->
      <ion-card>
        <ion-card-header>
          <ion-card-title>Notifications</ion-card-title>
        </ion-card-header>
        <ion-card-content>
          <ion-button expand="block" (click)="showToast()">
            <ion-icon name="notifications" slot="start"></ion-icon>
            Show Toast
          </ion-button>
        </ion-card-content>
      </ion-card>
    </ion-content>
  `,
  styles: [`
    .photo-preview {
      width: 100%;
      max-height: 300px;
      object-fit: cover;
      border-radius: 8px;
      margin-bottom: 16px;
    }
  `]
})
export class NativeFeaturesPage {
  photo: string | undefined;
  location: { latitude: number; longitude: number } | null = null;

  async takePicture() {
    try {
      const image = await Camera.getPhoto({
        quality: 90,
        allowEditing: false,
        resultType: CameraResultType.DataUrl,
        source: CameraSource.Camera
      });

      this.photo = image.dataUrl;
    } catch (error) {
      console.error('Error taking photo:', error);
    }
  }

  async selectFromGallery() {
    try {
      const image = await Camera.getPhoto({
        quality: 90,
        allowEditing: true,
        resultType: CameraResultType.DataUrl,
        source: CameraSource.Photos
      });

      this.photo = image.dataUrl;
    } catch (error) {
      console.error('Error selecting photo:', error);
    }
  }

  async getCurrentLocation() {
    try {
      const coordinates = await Geolocation.getCurrentPosition();
      this.location = {
        latitude: coordinates.coords.latitude,
        longitude: coordinates.coords.longitude
      };
    } catch (error) {
      console.error('Error getting location:', error);
      await this.showToast('Unable to get location');
    }
  }

  async shareContent() {
    try {
      await Share.share({
        title: 'Check this out!',
        text: 'Really awesome thing you need to see right now',
        url: 'https://example.com',
        dialogTitle: 'Share with friends'
      });
    } catch (error) {
      console.error('Error sharing:', error);
    }
  }

  async showToast(message: string = 'This is a native toast!') {
    await Toast.show({
      text: message,
      duration: 'short',
      position: 'bottom'
    });
  }
}
```

## Common Mistakes

### 1. Not Using IonRouterOutlet

**Wrong:**
```html
<router-outlet></router-outlet>
```

**Correct:**
```html
<ion-router-outlet></ion-router-outlet>
```

### 2. Missing IonicRouteStrategy

**Wrong:**
```typescript
@NgModule({
  providers: []
})
```

**Correct:**
```typescript
import { IonicRouteStrategy } from '@ionic/angular';

@NgModule({
  providers: [
    { provide: RouteReuseStrategy, useClass: IonicRouteStrategy }
  ]
})
```

### 3. Not Handling Platform-Specific Styles

**Wrong:**
```typescript
// Hardcoding iOS or Android specific code
```

**Correct:**
```typescript
import { Platform } from '@ionic/angular';

constructor(private platform: Platform) {
  if (this.platform.is('ios')) {
    // iOS-specific code
  } else if (this.platform.is('android')) {
    // Android-specific code
  }
}
```

### 4. Not Dismissing Modals/Loading Properly

**Wrong:**
```typescript
async showLoading() {
  const loading = await this.loadingCtrl.create();
  await loading.present();
  // Forgot to dismiss
}
```

**Correct:**
```typescript
async showLoading() {
  const loading = await this.loadingCtrl.create();
  await loading.present();
  
  // Do work
  
  await loading.dismiss();
}
```

## Best Practices

### 1. Use Lazy Loading for Pages

```typescript
const routes: Routes = [
  {
    path: 'home',
    loadChildren: () => import('./home/home.module').then(m => m.HomePageModule)
  }
];
```

### 2. Implement Proper Error Handling

```typescript
async fetchData() {
  const loading = await this.loadingCtrl.create();
  await loading.present();

  try {
    const data = await this.api.getData();
    // Handle success
  } catch (error) {
    const toast = await this.toastCtrl.create({
      message: 'Error loading data',
      duration: 2000,
      color: 'danger'
    });
    await toast.present();
  } finally {
    await loading.dismiss();
  }
}
```

### 3. Use Platform-Specific Configurations

```typescript
// src/theme/variables.scss
:root {
  --ion-color-primary: #3880ff;
}

.ios {
  --ion-color-primary: #007aff;
}

.md {
  --ion-color-primary: #3880ff;
}
```

### 4. Optimize Images and Assets

```typescript
// Use appropriate image sizes
<img [src]="image" loading="lazy" />
```

### 5. Implement Proper Memory Management

```typescript
export class MyPage implements OnDestroy {
  private destroy$ = new Subject();

  ngOnInit() {
    this.dataService.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => {
        // Handle data
      });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

## When to Use Ionic Angular

### Use When:

- Building cross-platform mobile applications (iOS, Android, Web)
- Need native device API access (camera, geolocation, etc.)
- Want single codebase for multiple platforms
- Building Progressive Web Apps (PWAs)
- Need mobile-optimized UI components
- Building hybrid mobile applications
- Want rapid mobile development

### Avoid When:

- Building desktop-only applications
- Need highly customized, non-standard mobile UI
- Require extremely high performance (use native)
- Building simple websites
- Need extensive platform-specific features (consider React Native or native)

## Interview Questions

### Q1: What is the difference between ion-router-outlet and router-outlet?

**Answer:**

**ion-router-outlet:**
- Ionic-specific router outlet
- Provides mobile navigation animations
- Supports navigation stack management
- Enables swipe-to-go-back gesture
- Caches pages for better performance
- Required for Ionic navigation patterns

**router-outlet:**
- Standard Angular router outlet
- No mobile-specific animations
- No navigation stack
- No gesture support

**Key point:** Always use `ion-router-outlet` in Ionic applications for proper mobile navigation experience.

### Q2: How does Ionic handle platform-specific styling?

**Answer:** Ionic automatically detects the platform and applies appropriate styles:

1. **Automatic Mode Detection:**
```typescript
// Automatically detects iOS or Material Design (Android)
```

2. **Platform CSS Classes:**
```css
.ios { /* iOS-specific styles */ }
.md { /* Material Design (Android) styles */ }
```

3. **Platform Service:**
```typescript
constructor(private platform: Platform) {
  if (this.platform.is('ios')) {
    // iOS-specific code
  }
}
```

4. **Mode Property:**
```html
<ion-button mode="ios">iOS Style</ion-button>
<ion-button mode="md">Material Design</ion-button>
```

### Q3: Explain the navigation pattern in Ionic Angular.

**Answer:** Ionic uses a navigation stack pattern:

1. **Navigation Stack:**
- Pages are pushed onto a stack
- Back button pops pages off the stack
- Maintains navigation history

2. **Navigation Methods:**
```typescript
// Navigate forward
this.router.navigate(['/page']);

// Navigate with animation direction
this.navCtrl.navigateForward('/page');

// Navigate back
this.navCtrl.navigateBack('/previous');

// Navigate to root
this.navCtrl.navigateRoot('/home');
```

3. **Route Reuse:**
- IonicRouteStrategy caches pages
- Faster navigation
- Preserves component state

4. **Gestures:**
- Swipe-to-go-back on iOS
- Hardware back button on Android

### Q4: How do you integrate native device features with Ionic?

**Answer:** Use Capacitor (recommended) or Cordova:

**Capacitor Example:**
```typescript
// Install plugin
npm install @capacitor/camera

// Import and use
import { Camera, CameraResultType } from '@capacitor/camera';

async takePicture() {
  const image = await Camera.getPhoto({
    quality: 90,
    resultType: CameraResultType.DataUrl,
    source: CameraSource.Camera
  });
}
```

**Available Plugins:**
- Camera
- Geolocation
- Filesystem
- Storage
- Push Notifications
- Device info
- Network
- And many more

**Platform-Specific Code:**
```typescript
import { Capacitor } from '@capacitor/core';

if (Capacitor.isNativePlatform()) {
  // Native mobile code
} else {
  // Web code
}
```

### Q5: What are the performance considerations for Ionic applications?

**Answer:**

1. **Lazy Loading:**
```typescript
// Load pages on demand
loadChildren: () => import('./page/page.module')
```

2. **Virtual Scrolling:**
```html
<ion-virtual-scroll [items]="items">
  <ion-item *virtualItem="let item">
    {{item}}
  </ion-item>
</ion-virtual-scroll>
```

3. **Image Optimization:**
```html
<img loading="lazy" />
```

4. **Change Detection:**
```typescript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
```

5. **Memory Management:**
```typescript
// Unsubscribe from observables
ngOnDestroy() {
  this.subscription.unsubscribe();
}
```

6. **Minimize Bundle Size:**
- Tree shaking
- Remove unused code
- Optimize dependencies

7. **Use Production Builds:**
```bash
ionic build --prod
```

## Key Takeaways

1. **Mobile-First**: Designed specifically for mobile applications
2. **Cross-Platform**: Single codebase for iOS, Android, and Web
3. **Native Access**: Capacitor provides access to native device APIs
4. **Platform Styling**: Automatic iOS and Android styling
5. **Navigation Stack**: Mobile-style navigation with gestures
6. **IonicRouteStrategy**: Caches pages for better performance
7. **Rich Components**: Comprehensive mobile-optimized UI components
8. **Gestures**: Built-in gesture support (swipe, pan, etc.)
9. **PWA Support**: Can be deployed as Progressive Web Apps
10. **Active Community**: Large community and extensive documentation
11. **Ionic CLI**: Powerful CLI for development and building
12. **Theming**: Easy customization with CSS variables
13. **Forms**: Rich form controls optimized for mobile
14. **Modals**: Flexible modal system with breakpoints
15. **Performance**: Optimized for mobile performance

## Resources

- **Official Documentation**: https://ionicframework.com/docs
- **GitHub Repository**: https://github.com/ionic-team/ionic-framework
- **Ionic Forum**: https://forum.ionicframework.com/
- **Capacitor Docs**: https://capacitorjs.com/docs
- **Ionic CLI**: https://ionicframework.com/docs/cli
- **Component Playground**: https://ionicframework.com/docs/components
- **Ionic Academy**: https://ionicacademy.com/
- **YouTube Channel**: https://www.youtube.com/c/Ionicframework
- **Blog**: https://ionicframework.com/blog
- **Discord Community**: https://ionic.link/discord