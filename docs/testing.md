# Testing Strategy

## Testing Philosophy

PhotoShare follows a comprehensive testing strategy that ensures reliability, performance, and security across all system components. Our testing approach implements the testing pyramid with emphasis on fast feedback loops and comprehensive coverage.

## Testing Pyramid

```
                    E2E Tests
                   /          \
                  /    UI      \
                 /              \
                /________________\
               /                  \
              /  Integration Tests \
             /                      \
            /________________________\
           /                          \
          /        Unit Tests          \
         /____________________________\
```

### Testing Distribution
- **Unit Tests**: 70% - Fast, isolated, comprehensive
- **Integration Tests**: 20% - Component interactions
- **End-to-End Tests**: 10% - Critical user journeys

## Unit Testing

### Backend Unit Tests (Node.js/Express)

#### Test Configuration
```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  setupFilesAfterEnv: ['<rootDir>/tests/setup.js'],
  testMatch: [
    '<rootDir>/src/**/__tests__/**/*.test.js',
    '<rootDir>/tests/unit/**/*.test.js'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/index.js'
  ],
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1'
  }
};
```

#### Service Layer Tests
```javascript
// tests/unit/services/photoService.test.js
const PhotoService = require('@/services/photoService');
const { mockS3, mockDatabase, mockCache } = require('../../mocks');

describe('PhotoService', () => {
  let photoService;
  
  beforeEach(() => {
    photoService = new PhotoService();
    jest.clearAllMocks();
  });
  
  describe('uploadPhoto', () => {
    it('should upload photo and save metadata', async () => {
      // Arrange
      const mockFile = {
        buffer: Buffer.from('fake-image-data'),
        mimetype: 'image/jpeg',
        originalname: 'test.jpg',
        size: 1024
      };
      const userId = 'user-123';
      const metadata = {
        title: 'Test Photo',
        description: 'A test photo'
      };
      
      mockS3.upload.mockResolvedValue({
        Location: 'https://s3.amazonaws.com/bucket/photo.jpg',
        Key: 'photos/user-123/photo.jpg'
      });
      
      mockDatabase.photos.create.mockResolvedValue({
        id: 'photo-123',
        userId,
        url: 'https://s3.amazonaws.com/bucket/photo.jpg',
        ...metadata
      });
      
      // Act
      const result = await photoService.uploadPhoto(mockFile, userId, metadata);
      
      // Assert
      expect(mockS3.upload).toHaveBeenCalledWith({
        Bucket: expect.any(String),
        Key: expect.stringContaining('photos/user-123'),
        Body: mockFile.buffer,
        ContentType: mockFile.mimetype
      });
      
      expect(mockDatabase.photos.create).toHaveBeenCalledWith({
        userId,
        filename: mockFile.originalname,
        size: mockFile.size,
        url: expect.any(String),
        ...metadata
      });
      
      expect(result).toMatchObject({
        id: 'photo-123',
        userId,
        url: expect.any(String)
      });
    });
    
    it('should handle S3 upload failure', async () => {
      // Arrange
      const mockFile = { buffer: Buffer.from('data'), mimetype: 'image/jpeg' };
      const error = new Error('S3 upload failed');
      mockS3.upload.mockRejectedValue(error);
      
      // Act & Assert
      await expect(
        photoService.uploadPhoto(mockFile, 'user-123', {})
      ).rejects.toThrow('Failed to upload photo');
      
      expect(mockDatabase.photos.create).not.toHaveBeenCalled();
    });
    
    it('should validate file type', async () => {
      // Arrange
      const invalidFile = {
        buffer: Buffer.from('data'),
        mimetype: 'text/plain'
      };
      
      // Act & Assert
      await expect(
        photoService.uploadPhoto(invalidFile, 'user-123', {})
      ).rejects.toThrow('Invalid file type');
    });
  });
  
  describe('getPhotoById', () => {
    it('should return photo with user details', async () => {
      // Arrange
      const photoId = 'photo-123';
      const mockPhoto = {
        id: photoId,
        userId: 'user-123',
        title: 'Test Photo',
        url: 'https://example.com/photo.jpg'
      };
      
      mockCache.get.mockResolvedValue(null);
      mockDatabase.photos.findById.mockResolvedValue(mockPhoto);
      
      // Act
      const result = await photoService.getPhotoById(photoId);
      
      // Assert
      expect(mockCache.get).toHaveBeenCalledWith(`photo:${photoId}`);
      expect(mockDatabase.photos.findById).toHaveBeenCalledWith(photoId);
      expect(mockCache.set).toHaveBeenCalledWith(
        `photo:${photoId}`,
        mockPhoto,
        3600
      );
      expect(result).toEqual(mockPhoto);
    });
    
    it('should return cached photo', async () => {
      // Arrange
      const photoId = 'photo-123';
      const cachedPhoto = { id: photoId, title: 'Cached Photo' };
      mockCache.get.mockResolvedValue(cachedPhoto);
      
      // Act
      const result = await photoService.getPhotoById(photoId);
      
      // Assert
      expect(mockDatabase.photos.findById).not.toHaveBeenCalled();
      expect(result).toEqual(cachedPhoto);
    });
  });
  
  describe('generateFeed', () => {
    it('should generate personalized feed', async () => {
      // Arrange
      const userId = 'user-123';
      const mockPhotos = [
        { id: 'photo-1', title: 'Photo 1' },
        { id: 'photo-2', title: 'Photo 2' }
      ];
      
      mockDatabase.photos.getPersonalizedFeed.mockResolvedValue(mockPhotos);
      
      // Act
      const result = await photoService.generateFeed(userId, 1, 20);
      
      // Assert
      expect(mockDatabase.photos.getPersonalizedFeed).toHaveBeenCalledWith({
        userId,
        page: 1,
        limit: 20
      });
      expect(result).toEqual(mockPhotos);
    });
  });
});
```

#### Controller Tests
```javascript
// tests/unit/controllers/photoController.test.js
const request = require('supertest');
const app = require('@/app');
const PhotoService = require('@/services/photoService');

jest.mock('@/services/photoService');

describe('Photo Controller', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  describe('POST /api/photos', () => {
    it('should upload photo successfully', async () => {
      // Arrange
      const mockPhoto = {
        id: 'photo-123',
        title: 'Test Photo',
        url: 'https://example.com/photo.jpg'
      };
      
      PhotoService.prototype.uploadPhoto = jest.fn().mockResolvedValue(mockPhoto);
      
      // Act
      const response = await request(app)
        .post('/api/photos')
        .set('Authorization', 'Bearer valid-token')
        .attach('photo', Buffer.from('fake-image'), 'test.jpg')
        .field('title', 'Test Photo')
        .field('description', 'Test Description');
      
      // Assert
      expect(response.status).toBe(201);
      expect(response.body).toMatchObject({
        success: true,
        data: mockPhoto
      });
    });
    
    it('should require authentication', async () => {
      // Act
      const response = await request(app)
        .post('/api/photos')
        .attach('photo', Buffer.from('fake-image'), 'test.jpg');
      
      // Assert
      expect(response.status).toBe(401);
      expect(response.body.error).toBe('Authentication required');
    });
    
    it('should validate file upload', async () => {
      // Act
      const response = await request(app)
        .post('/api/photos')
        .set('Authorization', 'Bearer valid-token')
        .field('title', 'Test Photo');
      
      // Assert
      expect(response.status).toBe(400);
      expect(response.body.error).toBe('No file uploaded');
    });
  });
  
  describe('GET /api/photos/:id', () => {
    it('should return photo details', async () => {
      // Arrange
      const mockPhoto = {
        id: 'photo-123',
        title: 'Test Photo',
        user: { id: 'user-123', username: 'testuser' }
      };
      
      PhotoService.prototype.getPhotoById = jest.fn().mockResolvedValue(mockPhoto);
      
      // Act
      const response = await request(app)
        .get('/api/photos/photo-123');
      
      // Assert
      expect(response.status).toBe(200);
      expect(response.body.data).toEqual(mockPhoto);
    });
    
    it('should handle photo not found', async () => {
      // Arrange
      PhotoService.prototype.getPhotoById = jest.fn().mockResolvedValue(null);
      
      // Act
      const response = await request(app)
        .get('/api/photos/nonexistent');
      
      // Assert
      expect(response.status).toBe(404);
      expect(response.body.error).toBe('Photo not found');
    });
  });
});
```

### Frontend Unit Tests (React)

#### Component Tests
```javascript
// src/components/__tests__/PhotoCard.test.jsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { Provider } from 'react-redux';
import PhotoCard from '../PhotoCard';
import { store } from '../../store';

const mockPhoto = {
  id: 'photo-123',
  title: 'Test Photo',
  description: 'Test Description',
  url: 'https://example.com/photo.jpg',
  user: {
    id: 'user-123',
    username: 'testuser',
    avatarUrl: 'https://example.com/avatar.jpg'
  },
  likeCount: 5,
  commentCount: 2,
  isLiked: false
};

const renderWithProvider = (component) => {
  return render(
    <Provider store={store}>
      {component}
    </Provider>
  );
};

describe('PhotoCard', () => {
  it('should render photo information', () => {
    renderWithProvider(<PhotoCard photo={mockPhoto} />);
    
    expect(screen.getByAltText('Test Photo')).toBeInTheDocument();
    expect(screen.getByText('Test Photo')).toBeInTheDocument();
    expect(screen.getByText('Test Description')).toBeInTheDocument();
    expect(screen.getByText('@testuser')).toBeInTheDocument();
    expect(screen.getByText('5 likes')).toBeInTheDocument();
    expect(screen.getByText('2 comments')).toBeInTheDocument();
  });
  
  it('should handle like button click', async () => {
    const mockOnLike = jest.fn();
    renderWithProvider(
      <PhotoCard photo={mockPhoto} onLike={mockOnLike} />
    );
    
    const likeButton = screen.getByRole('button', { name: /like/i });
    fireEvent.click(likeButton);
    
    await waitFor(() => {
      expect(mockOnLike).toHaveBeenCalledWith('photo-123');
    });
  });
  
  it('should show liked state', () => {
    const likedPhoto = { ...mockPhoto, isLiked: true };
    renderWithProvider(<PhotoCard photo={likedPhoto} />);
    
    const likeButton = screen.getByRole('button', { name: /unlike/i });
    expect(likeButton).toHaveClass('liked');
  });
  
  it('should handle image load error', () => {
    renderWithProvider(<PhotoCard photo={mockPhoto} />);
    
    const image = screen.getByAltText('Test Photo');
    fireEvent.error(image);
    
    expect(screen.getByText('Failed to load image')).toBeInTheDocument();
  });
});
```

#### Hook Tests
```javascript
// src/hooks/__tests__/usePhotoUpload.test.js
import { renderHook, act } from '@testing-library/react';
import { Provider } from 'react-redux';
import { store } from '../../store';
import usePhotoUpload from '../usePhotoUpload';

const wrapper = ({ children }) => (
  <Provider store={store}>{children}</Provider>
);

describe('usePhotoUpload', () => {
  it('should initialize with default state', () => {
    const { result } = renderHook(() => usePhotoUpload(), { wrapper });
    
    expect(result.current.isUploading).toBe(false);
    expect(result.current.progress).toBe(0);
    expect(result.current.error).toBe(null);
  });
  
  it('should handle file upload', async () => {
    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({
        success: true,
        data: { id: 'photo-123', url: 'https://example.com/photo.jpg' }
      })
    });
    
    const { result } = renderHook(() => usePhotoUpload(), { wrapper });
    
    const file = new File(['fake-image'], 'test.jpg', { type: 'image/jpeg' });
    const metadata = { title: 'Test Photo', description: 'Test Description' };
    
    await act(async () => {
      await result.current.uploadPhoto(file, metadata);
    });
    
    expect(result.current.isUploading).toBe(false);
    expect(result.current.error).toBe(null);
  });
  
  it('should handle upload error', async () => {
    global.fetch = jest.fn().mockRejectedValue(new Error('Upload failed'));
    
    const { result } = renderHook(() => usePhotoUpload(), { wrapper });
    
    const file = new File(['fake-image'], 'test.jpg', { type: 'image/jpeg' });
    
    await act(async () => {
      await result.current.uploadPhoto(file, {});
    });
    
    expect(result.current.isUploading).toBe(false);
    expect(result.current.error).toBe('Upload failed');
  });
});
```

## Integration Testing

### API Integration Tests
```javascript
// tests/integration/api.test.js
const request = require('supertest');
const app = require('@/app');
const { setupTestDB, teardownTestDB } = require('../helpers/database');
const { createTestUser, generateToken } = require('../helpers/auth');

describe('Photo API Integration', () => {
  let testUser;
  let authToken;
  
  beforeAll(async () => {
    await setupTestDB();
    testUser = await createTestUser();
    authToken = generateToken(testUser.id);
  });
  
  afterAll(async () => {
    await teardownTestDB();
  });
  
  describe('Photo Upload Flow', () => {
    it('should complete full photo upload and retrieval flow', async () => {
      // Upload photo
      const uploadResponse = await request(app)
        .post('/api/photos')
        .set('Authorization', `Bearer ${authToken}`)
        .attach('photo', Buffer.from('fake-image-data'), 'test.jpg')
        .field('title', 'Integration Test Photo')
        .field('description', 'Test photo description');
      
      expect(uploadResponse.status).toBe(201);
      expect(uploadResponse.body.success).toBe(true);
      
      const photoId = uploadResponse.body.data.id;
      
      // Retrieve photo
      const getResponse = await request(app)
        .get(`/api/photos/${photoId}`);
      
      expect(getResponse.status).toBe(200);
      expect(getResponse.body.data).toMatchObject({
        id: photoId,
        title: 'Integration Test Photo',
        description: 'Test photo description',
        user: {
          id: testUser.id,
          username: testUser.username
        }
      });
      
      // Like photo
      const likeResponse = await request(app)
        .post(`/api/photos/${photoId}/like`)
        .set('Authorization', `Bearer ${authToken}`);
      
      expect(likeResponse.status).toBe(200);
      
      // Verify like count
      const updatedPhoto = await request(app)
        .get(`/api/photos/${photoId}`);
      
      expect(updatedPhoto.body.data.likeCount).toBe(1);
      expect(updatedPhoto.body.data.isLiked).toBe(true);
    });
  });
  
  describe('Feed Generation', () => {
    it('should generate personalized feed', async () => {
      // Create another user and photos
      const otherUser = await createTestUser('otheruser@example.com');
      const otherToken = generateToken(otherUser.id);
      
      // Upload photos from other user
      await request(app)
        .post('/api/photos')
        .set('Authorization', `Bearer ${otherToken}`)
        .attach('photo', Buffer.from('photo1'), 'photo1.jpg')
        .field('title', 'Feed Test Photo 1');
      
      await request(app)
        .post('/api/photos')
        .set('Authorization', `Bearer ${otherToken}`)
        .attach('photo', Buffer.from('photo2'), 'photo2.jpg')
        .field('title', 'Feed Test Photo 2');
      
      // Follow other user
      await request(app)
        .post(`/api/users/${otherUser.id}/follow`)
        .set('Authorization', `Bearer ${authToken}`);
      
      // Get feed
      const feedResponse = await request(app)
        .get('/api/feed')
        .set('Authorization', `Bearer ${authToken}`);
      
      expect(feedResponse.status).toBe(200);
      expect(feedResponse.body.data.length).toBeGreaterThan(0);
      expect(feedResponse.body.data[0]).toHaveProperty('title');
      expect(feedResponse.body.data[0]).toHaveProperty('user');
    });
  });
});
```

### Database Integration Tests
```javascript
// tests/integration/database.test.js
const { Pool } = require('pg');
const PhotoRepository = require('@/repositories/photoRepository');
const UserRepository = require('@/repositories/userRepository');

describe('Database Integration', () => {
  let pool;
  let photoRepo;
  let userRepo;
  
  beforeAll(async () => {
    pool = new Pool({
      connectionString: process.env.TEST_DATABASE_URL
    });
    
    photoRepo = new PhotoRepository(pool);
    userRepo = new UserRepository(pool);
  });
  
  afterAll(async () => {
    await pool.end();
  });
  
  beforeEach(async () => {
    // Clean up before each test
    await pool.query('TRUNCATE TABLE photos, users CASCADE');
  });
  
  describe('Photo Repository', () => {
    it('should create and retrieve photo', async () => {
      // Create user first
      const user = await userRepo.create({
        email: 'test@example.com',
        username: 'testuser',
        passwordHash: 'hashed-password'
      });
      
      // Create photo
      const photoData = {
        userId: user.id,
        title: 'Test Photo',
        description: 'Test Description',
        filename: 'test.jpg',
        url: 'https://example.com/test.jpg',
        size: 1024
      };
      
      const photo = await photoRepo.create(photoData);
      
      expect(photo).toMatchObject({
        id: expect.any(String),
        ...photoData,
        createdAt: expect.any(Date),
        updatedAt: expect.any(Date)
      });
      
      // Retrieve photo
      const retrievedPhoto = await photoRepo.findById(photo.id);
      expect(retrievedPhoto).toMatchObject(photo);
    });
    
    it('should handle foreign key constraints', async () => {
      // Try to create photo with non-existent user
      const photoData = {
        userId: 'non-existent-user',
        title: 'Test Photo',
        filename: 'test.jpg',
        url: 'https://example.com/test.jpg'
      };
      
      await expect(photoRepo.create(photoData)).rejects.toThrow();
    });
  });
  
  describe('Complex Queries', () => {
    it('should generate accurate feed with joins', async () => {
      // Create multiple users
      const user1 = await userRepo.create({
        email: 'user1@example.com',
        username: 'user1'
      });
      
      const user2 = await userRepo.create({
        email: 'user2@example.com',
        username: 'user2'
      });
      
      // Create photos
      const photo1 = await photoRepo.create({
        userId: user1.id,
        title: 'Photo 1',
        filename: 'photo1.jpg',
        url: 'https://example.com/photo1.jpg'
      });
      
      const photo2 = await photoRepo.create({
        userId: user2.id,
        title: 'Photo 2',
        filename: 'photo2.jpg',
        url: 'https://example.com/photo2.jpg'
      });
      
      // Create follows relationship
      await pool.query(
        'INSERT INTO follows (follower_id, followed_id) VALUES ($1, $2)',
        [user1.id, user2.id]
      );
      
      // Generate feed
      const feed = await photoRepo.getPersonalizedFeed({
        userId: user1.id,
        limit: 10,
        offset: 0
      });
      
      expect(feed.length).toBe(2); // user1's own photo + user2's photo
      expect(feed).toEqual(
        expect.arrayContaining([
          expect.objectContaining({
            id: photo1.id,
            user: expect.objectContaining({ username: 'user1' })
          }),
          expect.objectContaining({
            id: photo2.id,
            user: expect.objectContaining({ username: 'user2' })
          })
        ])
      );
    });
  });
});
```

### AWS Services Integration Tests
```javascript
// tests/integration/aws.test.js
const AWS = require('aws-sdk');
const S3Service = require('@/services/s3Service');
const { LocalStack } = require('@localstack/node');

describe('AWS Services Integration', () => {
  let localstack;
  let s3Service;
  
  beforeAll(async () => {
    // Start LocalStack for testing
    localstack = new LocalStack();
    await localstack.start();
    
    // Configure AWS SDK for LocalStack
    AWS.config.update({
      endpoint: 'http://localhost:4566',
      accessKeyId: 'test',
      secretAccessKey: 'test',
      region: 'us-east-1'
    });
    
    s3Service = new S3Service();
    
    // Create test bucket
    await s3Service.createBucket('test-photos-bucket');
  });
  
  afterAll(async () => {
    await localstack.stop();
  });
  
  describe('S3 Integration', () => {
    it('should upload and retrieve file', async () => {
      const fileContent = Buffer.from('test file content');
      const key = 'test-photos/test-image.jpg';
      
      // Upload file
      const uploadResult = await s3Service.uploadFile({
        bucket: 'test-photos-bucket',
        key,
        body: fileContent,
        contentType: 'image/jpeg'
      });
      
      expect(uploadResult).toHaveProperty('Location');
      expect(uploadResult.Key).toBe(key);
      
      // Retrieve file
      const retrievedFile = await s3Service.getFile({
        bucket: 'test-photos-bucket',
        key
      });
      
      expect(retrievedFile.Body).toEqual(fileContent);
      expect(retrievedFile.ContentType).toBe('image/jpeg');
    });
    
    it('should generate signed URLs', async () => {
      const key = 'test-photos/signed-url-test.jpg';
      
      // Upload file first
      await s3Service.uploadFile({
        bucket: 'test-photos-bucket',
        key,
        body: Buffer.from('test content'),
        contentType: 'image/jpeg'
      });
      
      // Generate signed URL
      const signedUrl = await s3Service.getSignedUrl({
        bucket: 'test-photos-bucket',
        key,
        expires: 3600
      });
      
      expect(signedUrl).toMatch(/^http:\/\/localhost:4566/);
      expect(signedUrl).toContain(key);
    });
  });
  
  describe('DynamoDB Integration', () => {
    it('should store and retrieve user sessions', async () => {
      const dynamodb = new AWS.DynamoDB.DocumentClient();
      
      // Create table
      const dynamoDbClient = new AWS.DynamoDB();
      await dynamoDbClient.createTable({
        TableName: 'user-sessions',
        KeySchema: [
          { AttributeName: 'userId', KeyType: 'HASH' }
        ],
        AttributeDefinitions: [
          { AttributeName: 'userId', AttributeType: 'S' }
        ],
        BillingMode: 'PAY_PER_REQUEST'
      }).promise();
      
      // Put item
      const sessionData = {
        userId: 'user-123',
        sessionId: 'session-456',
        createdAt: Date.now(),
        expiresAt: Date.now() + 3600000
      };
      
      await dynamodb.put({
        TableName: 'user-sessions',
        Item: sessionData
      }).promise();
      
      // Get item
      const result = await dynamodb.get({
        TableName: 'user-sessions',
        Key: { userId: 'user-123' }
      }).promise();
      
      expect(result.Item).toMatchObject(sessionData);
    });
  });
});
```

## End-to-End Testing

### E2E Test Configuration
```javascript
// e2e/playwright.config.js
const { defineConfig, devices } = require('@playwright/test');

module.exports = defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] }
    }
  ],
  webServer: {
    command: 'npm run start:test',
    port: 3000,
    reuseExistingServer: !process.env.CI
  }
});
```

### Critical User Journey Tests
```javascript
// e2e/tests/user-journey.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Photo Sharing User Journey', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });
  
  test('complete user registration and photo upload flow', async ({ page }) => {
    // Register new user
    await page.click('[data-testid="register-button"]');
    await page.fill('[data-testid="email-input"]', 'e2e-test@example.com');
    await page.fill('[data-testid="username-input"]', 'e2euser');
    await page.fill('[data-testid="password-input"]', 'TestPassword123!');
    await page.fill('[data-testid="confirm-password-input"]', 'TestPassword123!');
    await page.click('[data-testid="submit-registration"]');
    
    // Verify registration success
    await expect(page.locator('[data-testid="welcome-message"]')).toBeVisible();
    
    // Upload photo
    await page.click('[data-testid="upload-button"]');
    await page.setInputFiles('[data-testid="file-input"]', 'e2e/fixtures/test-photo.jpg');
    await page.fill('[data-testid="title-input"]', 'My E2E Test Photo');
    await page.fill('[data-testid="description-input"]', 'This is a test photo uploaded via E2E test');
    await page.selectOption('[data-testid="privacy-select"]', 'public');
    await page.click('[data-testid="upload-submit"]');
    
    // Verify upload success
    await expect(page.locator('[data-testid="upload-success"]')).toBeVisible();
    await expect(page.locator('text=My E2E Test Photo')).toBeVisible();
    
    // Navigate to profile
    await page.click('[data-testid="profile-link"]');
    await expect(page.locator('[data-testid="photo-grid"]')).toContainText('My E2E Test Photo');
    
    // Test photo interactions
    await page.click('[data-testid="photo-card"]:first-child');
    await page.click('[data-testid="like-button"]');
    await expect(page.locator('[data-testid="like-count"]')).toContainText('1');
    
    // Add comment
    await page.fill('[data-testid="comment-input"]', 'Great photo!');
    await page.click('[data-testid="comment-submit"]');
    await expect(page.locator('[data-testid="comment-list"]')).toContainText('Great photo!');
  });
  
  test('photo feed and discovery flow', async ({ page, context }) => {
    // Login as existing user
    await page.click('[data-testid="login-button"]');
    await page.fill('[data-testid="email-input"]', 'testuser@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-submit"]');
    
    // Navigate to feed
    await page.click('[data-testid="feed-link"]');
    await expect(page.locator('[data-testid="photo-feed"]')).toBeVisible();
    
    // Test infinite scroll
    const initialPhotoCount = await page.locator('[data-testid="photo-card"]').count();
    await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));
    await page.waitForTimeout(2000); // Wait for new photos to load
    
    const newPhotoCount = await page.locator('[data-testid="photo-card"]').count();
    expect(newPhotoCount).toBeGreaterThan(initialPhotoCount);
    
    // Test search functionality
    await page.fill('[data-testid="search-input"]', 'landscape');
    await page.press('[data-testid="search-input"]', 'Enter');
    await expect(page.locator('[data-testid="search-results"]')).toBeVisible();
    
    // Filter search results
    await page.click('[data-testid="filter-recent"]');
    await page.waitForLoadState('networkidle');
    
    // Verify filtered results
    const searchResults = page.locator('[data-testid="photo-card"]');
    expect(await searchResults.count()).toBeGreaterThan(0);
  });
  
  test('responsive design on mobile', async ({ page }) => {
    // Set mobile viewport
    await page.setViewportSize({ width: 375, height: 667 });
    
    // Test mobile navigation
    await page.click('[data-testid="mobile-menu-button"]');
    await expect(page.locator('[data-testid="mobile-menu"]')).toBeVisible();
    
    // Test mobile photo upload
    await page.click('[data-testid="mobile-upload-button"]');
    await page.setInputFiles('[data-testid="file-input"]', 'e2e/fixtures/mobile-test.jpg');
    
    // Verify mobile-optimized upload UI
    await expect(page.locator('[data-testid="mobile-upload-preview"]')).toBeVisible();
    await expect(page.locator('[data-testid="mobile-crop-tool"]')).toBeVisible();
    
    // Test swipe gestures on photo carousel
    const carousel = page.locator('[data-testid="photo-carousel"]');
    await carousel.hover();
    await page.mouse.down();
    await page.mouse.move(200, 0); // Swipe left
    await page.mouse.up();
    
    // Verify carousel navigation
    await expect(page.locator('[data-testid="carousel-indicator"].active')).toHaveAttribute('data-index', '1');
  });
});
```

### Performance Testing
```javascript
// e2e/tests/performance.spec.js
const { test, expect } = require('@playwright/test');

test.describe('Performance Tests', () => {
  test('page load performance', async ({ page }) => {
    const startTime = Date.now();
    
    await page.goto('/', { waitUntil: 'networkidle' });
    
    const loadTime = Date.now() - startTime;
    expect(loadTime).toBeLessThan(3000); // Page should load within 3 seconds
    
    // Check Core Web Vitals
    const metrics = await page.evaluate(() => {
      return new Promise((resolve) => {
        new PerformanceObserver((list) => {
          const entries = list.getEntries();
          resolve(entries.map(entry => ({
            name: entry.name,
            value: entry.value
          })));
        }).observe({ entryTypes: ['largest-contentful-paint', 'first-input-delay'] });
        
        // Fallback timeout
        setTimeout(() => resolve([]), 5000);
      });
    });
    
    const lcp = metrics.find(m => m.name === 'largest-contentful-paint');
    if (lcp) {
      expect(lcp.value).toBeLessThan(2500); // LCP should be under 2.5s
    }
  });
  
  test('photo upload performance', async ({ page }) => {
    await page.goto('/upload');
    
    // Login first
    await page.fill('[data-testid="email-input"]', 'testuser@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-submit"]');
    
    const uploadStartTime = Date.now();
    
    // Upload large image
    await page.setInputFiles('[data-testid="file-input"]', 'e2e/fixtures/large-photo.jpg');
    await page.fill('[data-testid="title-input"]', 'Performance Test Photo');
    await page.click('[data-testid="upload-submit"]');
    
    // Wait for upload completion
    await expect(page.locator('[data-testid="upload-success"]')).toBeVisible({ timeout: 30000 });
    
    const uploadTime = Date.now() - uploadStartTime;
    expect(uploadTime).toBeLessThan(15000); // Upload should complete within 15 seconds
  });
  
  test('feed scroll performance', async ({ page }) => {
    await page.goto('/feed');
    
    // Measure scroll performance
    const scrollMetrics = await page.evaluate(async () => {
      const measurements = [];
      
      for (let i = 0; i < 5; i++) {
        const startTime = performance.now();
        
        window.scrollBy(0, window.innerHeight);
        await new Promise(resolve => setTimeout(resolve, 500)); // Wait for content load
        
        const endTime = performance.now();
        measurements.push(endTime - startTime);
      }
      
      return measurements;
    });
    
    const avgScrollTime = scrollMetrics.reduce((a, b) => a + b, 0) / scrollMetrics.length;
    expect(avgScrollTime).toBeLessThan(100); // Average scroll time should be under 100ms
  });
});
```

## Load Testing

### K6 Load Tests
```javascript
// k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export let options = {
  stages: [
    { duration: '2m', target: 100 }, // Ramp up to 100 users
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 200 }, // Ramp up to 200 users
    { duration: '5m', target: 200 }, // Stay at 200 users
    { duration: '2m', target: 0 },   // Ramp down to 0 users
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500'], // 95% of requests must complete below 500ms
    'errors': ['rate<0.1'], // Error rate must be below 10%
  },
};

const BASE_URL = 'https://api.photoshare.com';

export function setup() {
  // Create test user and get auth token
  const loginResponse = http.post(`${BASE_URL}/api/auth/login`, {
    email: 'loadtest@example.com',
    password: 'LoadTest123!'
  });
  
  return { authToken: loginResponse.json('token') };
}

export default function(data) {
  const headers = {
    'Authorization': `Bearer ${data.authToken}`,
    'Content-Type': 'application/json'
  };
  
  // Test scenarios with different weights
  const scenario = Math.random();
  
  if (scenario < 0.4) {
    // 40% - Browse feed
    browseFeed(headers);
  } else if (scenario < 0.7) {
    // 30% - View photos
    viewPhoto(headers);
  } else if (scenario < 0.9) {
    // 20% - Like/comment on photos
    interactWithPhoto(headers);
  } else {
    // 10% - Upload photos
    uploadPhoto(headers);
  }
  
  sleep(1);
}

function browseFeed(headers) {
  const response = http.get(`${BASE_URL}/api/feed?page=1&limit=20`, { headers });
  
  check(response, {
    'feed loaded successfully': (r) => r.status === 200,
    'feed has photos': (r) => JSON.parse(r.body).data.length > 0,
  }) || errorRate.add(1);
}

function viewPhoto(headers) {
  // Get a random photo ID from feed first
  const feedResponse = http.get(`${BASE_URL}/api/feed?page=1&limit=5`, { headers });
  const photos = JSON.parse(feedResponse.body).data;
  
  if (photos.length > 0) {
    const randomPhoto = photos[Math.floor(Math.random() * photos.length)];
    const response = http.get(`${BASE_URL}/api/photos/${randomPhoto.id}`, { headers });
    
    check(response, {
      'photo loaded successfully': (r) => r.status === 200,
      'photo has details': (r) => JSON.parse(r.body).data.title !== undefined,
    }) || errorRate.add(1);
  }
}

function interactWithPhoto(headers) {
  const feedResponse = http.get(`${BASE_URL}/api/feed?page=1&limit=5`, { headers });
  const photos = JSON.parse(feedResponse.body).data;
  
  if (photos.length > 0) {
    const randomPhoto = photos[Math.floor(Math.random() * photos.length)];
    
    // Like photo
    const likeResponse = http.post(`${BASE_URL}/api/photos/${randomPhoto.id}/like`, {}, { headers });
    
    check(likeResponse, {
      'like successful': (r) => r.status === 200,
    }) || errorRate.add(1);
    
    // Add comment
    const commentResponse = http.post(`${BASE_URL}/api/photos/${randomPhoto.id}/comments`, 
      JSON.stringify({ content: 'Great photo! #loadtest' }), 
      { headers }
    );
    
    check(commentResponse, {
      'comment added successfully': (r) => r.status === 201,
    }) || errorRate.add(1);
  }
}

function uploadPhoto(headers) {
  // Simulate photo upload (simplified)
  const formData = {
    title: 'Load Test Photo',
    description: 'Photo uploaded during load testing',
    file: 'fake-image-data'
  };
  
  const response = http.post(`${BASE_URL}/api/photos`, formData, { headers });
  
  check(response, {
    'photo uploaded successfully': (r) => r.status === 201,
  }) || errorRate.add(1);
}

export function teardown(data) {
  console.log('Load test completed');
}
```

## Test Automation & CI/CD Integration

### GitHub Actions Test Workflow
```yaml
# .github/workflows/test.yml
name: Test Suite
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run unit tests
        run: npm run test:unit -- --coverage
        
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: photoshare_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run database migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/photoshare_test
          
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/photoshare_test
          REDIS_URL: redis://localhost:6379
          
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Install Playwright browsers
        run: npx playwright install --with-deps
        
      - name: Start test environment
        run: |
          npm run build
          npm run start:test &
          sleep 30
          
      - name: Run E2E tests
        run: npx playwright test
        
      - name: Upload E2E test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          
  load-tests:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup K6
        uses: grafana/setup-k6-action@v1
        
      - name: Run load tests
        run: k6 run k6/load-test.js
        env:
          BASE_URL: https://staging-api.photoshare.com
```

This comprehensive testing strategy ensures PhotoShare maintains high quality, performance, and reliability across all components while providing fast feedback to developers and maintaining confidence in deployments.