# Phase 4B: Frontend Feedback Integration

**Assigned to:** AI Worker (Frontend Developer)
**Model:** Claude Sonnet 4.5
**Environment:** encore_frontend directory
**Prerequisites:**
- Phase 1 completed (Foundation setup)
- Phase 2B completed (Authentication working)
- Phase 3B completed (Settings integration working)
- Phase 4A completed (Backend feedback API working)

## 📋 Overview

Integrate the **Feedback Management** system into the React frontend. This includes creating feedback list and detail pages, connecting to real backend API, implementing filtering and search, and maintaining the mock API toggle for development.

## ⚠️ CRITICAL: Understanding the Data Architecture

The backend serves feedback data from **two sources**:

### Data Sources

**1. MySQL Legacy Database (Read-Only via Backend):**
- Historical customer feedback/comments
- Customer names, emails, phone numbers
- Venue information
- Ratings (note: 99.59% are 0 in production data)
- Created dates

**2. DynamoDB (Read/Write via Backend):**
- Feedback processing status (pending, reviewed, sent, etc.)
- AI-generated responses
- Sentiment analysis results
- Action history and timestamps
- Email send tracking

### What the Frontend Receives

The backend **combines** both sources into a single unified `Feedback` object:

```typescript
interface Feedback {
  // From MySQL (via backend):
  id: number;                    // iddatasession from comment table
  venue_id: string;              // idclient from datasession
  venue_name?: string;           // description from client table
  customer_email: string;        // emailaddress from comment table
  customer_name?: string;        // Combined from eclub.firstname + lastname
  rating?: number;               // overall_rating from comment table (99.59% are 0!)
  comment: string;               // comment from comment table
  created_at: string;            // dateentered from comment table
  
  // From DynamoDB (via backend):
  status: FeedbackStatus;        // Current processing status
  sentiment?: Sentiment;         // AI sentiment analysis
  ai_response_id?: string;       // ID of AI response (if generated)
  last_updated?: string;         // Last status update time
  last_action_by?: string;       // User who last acted
  response_count: number;        // Times response regenerated
  email_sent_count: number;      // Times email sent
}
```

### Frontend Developer Notes

1. **You never directly query MySQL** - the backend handles this
2. **All API calls go through `/api/feedback` endpoints**
3. **The backend joins 4 MySQL tables** (comment, datasession, client, eclub) and enriches with DynamoDB status
4. **Customer names may be NULL** - handle gracefully in UI
5. **Ratings are mostly 0** - consider hiding if 0
6. **Legacy replies** exist in MySQL `reply_details` field but aren't used in new system

## 🎯 Objectives

1. Create feedback types (matching backend models)
2. Create feedback API service layer
3. Update Feedback List page with real API
4. Update Feedback Detail page with real API
5. Implement filtering, search, and pagination
6. Add status management UI
7. Maintain mock API toggle functionality
8. Test complete feedback flow

## 📁 Directory Structure

```
encore_frontend/
├── src/
│   ├── api/
│   │   └── feedbackApi.ts           # ← CREATE
│   ├── types/
│   │   ├── feedback.ts               # ← CREATE
│   │   └── feedbackResponse.ts      # ← CREATE
│   ├── pages/
│   │   ├── FeedbackList.tsx          # ← UPDATE (connect to real API)
│   │   └── FeedbackDetail.tsx        # ← UPDATE (connect to real API)
│   ├── components/
│   │   ├── FeedbackFilters.tsx       # ← CREATE (filter UI)
│   │   ├── FeedbackStatusBadge.tsx   # ← CREATE (status display)
│   │   └── Pagination.tsx            # ← CREATE (pagination component)
│   ├── hooks/
│   │   ├── useFeedbackList.ts        # ← CREATE (list hook)
│   │   └── useFeedbackDetail.ts      # ← CREATE (detail hook)
│   └── mockData/
│       └── feedbackData.ts           # ← KEEP (for mock mode)
```

## ✅ Task Checklist

### Task 1: Create Feedback Types (`src/types/feedback.ts`)

**Purpose:** TypeScript interfaces matching backend models

**Create file:** `src/types/feedback.ts`

```typescript
/**
 * Feedback type definitions matching backend models
 */

export enum FeedbackStatus {
  PENDING = 'pending',
  REVIEWED = 'reviewed',
  GENERATED = 'generated',
  EDITED = 'edited',
  SENT = 'sent',
  ARCHIVED = 'archived',
  SPAM = 'spam',
}

export enum Sentiment {
  POSITIVE = 'positive',
  NEUTRAL = 'neutral',
  NEGATIVE = 'negative',
  MIXED = 'mixed',
}

export interface Feedback {
  // From MySQL (read-only)
  id: number;
  venue_id: string;
  venue_name?: string;
  customer_email: string;
  customer_name?: string;
  rating?: number; // 1-5 stars
  comment: string;
  created_at: string;

  // From DynamoDB (writable)
  status: FeedbackStatus;
  sentiment?: Sentiment;
  ai_response_id?: string;
  last_updated?: string;
  last_action_by?: string;

  // Metadata
  response_count: number;
  email_sent_count: number;
}

export interface FeedbackListItem {
  id: number;
  venue_id: string;
  venue_name?: string;
  customer_name?: string;
  rating?: number;
  comment_preview: string; // First 100 chars
  status: FeedbackStatus;
  sentiment?: Sentiment;
  created_at: string;
  has_response: boolean;
}

export interface FeedbackFilter {
  venue_id?: string;
  status?: FeedbackStatus;
  sentiment?: Sentiment;
  rating?: number;
  date_from?: string; // ISO date string
  date_to?: string;
  search_text?: string;

  // Pagination
  page: number;
  page_size: number;
}

export interface FeedbackListResponse {
  items: FeedbackListItem[];
  total: number;
  page: number;
  page_size: number;
  total_pages: number;
}

export interface UpdateFeedbackStatusRequest {
  status: FeedbackStatus;
  notes?: string;
}

// Helper functions
export const getStatusColor = (status: FeedbackStatus): string => {
  const colorMap: Record<FeedbackStatus, string> = {
    [FeedbackStatus.PENDING]: '#f59e0b',      // orange
    [FeedbackStatus.REVIEWED]: '#3b82f6',     // blue
    [FeedbackStatus.GENERATED]: '#8b5cf6',    // purple
    [FeedbackStatus.EDITED]: '#6366f1',       // indigo
    [FeedbackStatus.SENT]: '#10b981',         // green
    [FeedbackStatus.ARCHIVED]: '#6b7280',     // gray
    [FeedbackStatus.SPAM]: '#ef4444',         // red
  };
  return colorMap[status] || '#6b7280';
};

export const getSentimentColor = (sentiment: Sentiment): string => {
  const colorMap: Record<Sentiment, string> = {
    [Sentiment.POSITIVE]: '#10b981',   // green
    [Sentiment.NEUTRAL]: '#6b7280',    // gray
    [Sentiment.NEGATIVE]: '#ef4444',   // red
    [Sentiment.MIXED]: '#f59e0b',      // orange
  };
  return colorMap[sentiment] || '#6b7280';
};

export const getStatusLabel = (status: FeedbackStatus): string => {
  const labels: Record<FeedbackStatus, string> = {
    [FeedbackStatus.PENDING]: 'Pending',
    [FeedbackStatus.REVIEWED]: 'Reviewed',
    [FeedbackStatus.GENERATED]: 'Generated',
    [FeedbackStatus.EDITED]: 'Edited',
    [FeedbackStatus.SENT]: 'Sent',
    [FeedbackStatus.ARCHIVED]: 'Archived',
    [FeedbackStatus.SPAM]: 'Spam',
  };
  return labels[status] || status;
};
```

---

### Task 2: Create Feedback Response Types (`src/types/feedbackResponse.ts`)

**Purpose:** Types for AI-generated responses

**Create file:** `src/types/feedbackResponse.ts`

```typescript
/**
 * Feedback response types (AI-generated responses)
 */

export enum ResponseGenerationMethod {
  AI_GENERATED = 'ai_generated',
  AI_REGENERATED = 'ai_regenerated',
  MANUAL = 'manual',
}

export interface FeedbackResponse {
  response_id: string;
  feedback_id: number;
  venue_id: string;

  // Response content
  response_text: string;
  generation_method: ResponseGenerationMethod;

  // AI generation details
  ai_model?: string;
  ai_temperature?: number;
  ai_max_tokens?: number;
  custom_instructions?: string;

  // Customer profile
  customer_profile?: Record<string, any>;

  // Metadata
  created_at: string;
  created_by: string;
  edited_at?: string;
  edited_by?: string;

  // Email sending
  email_sent: boolean;
  email_sent_at?: string;
  email_status?: string;
}

export interface CreateResponseRequest {
  response_text: string;
  generation_method?: ResponseGenerationMethod;
  custom_instructions?: string;
}

export interface RegenerateResponseRequest {
  custom_instructions: string;
  temperature?: number;
  max_tokens?: number;
}
```

---

### Task 3: Create Feedback API Service (`src/api/feedbackApi.ts`)

**Purpose:** API calls to backend feedback endpoints

**Create file:** `src/api/feedbackApi.ts`

```typescript
/**
 * Feedback API service
 * Handles all feedback-related API calls
 */
import apiClient from './apiClient';
import {
  Feedback,
  FeedbackListResponse,
  FeedbackFilter,
  UpdateFeedbackStatusRequest,
  FeedbackStatus,
} from '../types/feedback';
import { mockFeedbackList, mockFeedbackDetail } from '../mockData/feedbackData';
import { API_CONFIG } from '../config/api';

class FeedbackApi {
  /**
   * List feedback with filters and pagination
   */
  async listFeedback(filters: Partial<FeedbackFilter>): Promise<FeedbackListResponse> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Listing feedback with filters:', filters);
      // Simulate API delay
      await new Promise((resolve) => setTimeout(resolve, 600));

      // Filter mock data
      let items = mockFeedbackList.items;

      // Apply filters
      if (filters.venue_id) {
        items = items.filter((item) => item.venue_id === filters.venue_id);
      }
      if (filters.status) {
        items = items.filter((item) => item.status === filters.status);
      }
      if (filters.sentiment) {
        items = items.filter((item) => item.sentiment === filters.sentiment);
      }
      if (filters.rating) {
        items = items.filter((item) => item.rating === filters.rating);
      }
      if (filters.search_text) {
        const search = filters.search_text.toLowerCase();
        items = items.filter(
          (item) =>
            item.comment_preview.toLowerCase().includes(search) ||
            item.customer_name?.toLowerCase().includes(search)
        );
      }

      // Pagination
      const page = filters.page || 1;
      const page_size = filters.page_size || 20;
      const start = (page - 1) * page_size;
      const end = start + page_size;
      const paginatedItems = items.slice(start, end);

      return {
        items: paginatedItems,
        total: items.length,
        page,
        page_size,
        total_pages: Math.ceil(items.length / page_size),
      };
    }

    // Real API call
    const params: any = {
      ...filters,
      page: filters.page || 1,
      page_size: filters.page_size || 20,
    };

    const response = await apiClient.get<FeedbackListResponse>('/api/feedback', {
      params,
    });
    return response.data;
  }

  /**
   * Get single feedback by ID
   */
  async getFeedback(feedbackId: number): Promise<Feedback> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Getting feedback:', feedbackId);
      await new Promise((resolve) => setTimeout(resolve, 400));

      const mock = mockFeedbackDetail[feedbackId];
      if (!mock) {
        throw new Error(`Feedback ${feedbackId} not found`);
      }
      return mock;
    }

    const response = await apiClient.get<Feedback>(`/api/feedback/${feedbackId}`);
    return response.data;
  }

  /**
   * Update feedback status
   */
  async updateStatus(
    feedbackId: number,
    request: UpdateFeedbackStatusRequest
  ): Promise<{ success: boolean; message: string }> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Updating status:', feedbackId, request);
      await new Promise((resolve) => setTimeout(resolve, 500));

      // Update mock data
      if (mockFeedbackDetail[feedbackId]) {
        mockFeedbackDetail[feedbackId].status = request.status;
        mockFeedbackDetail[feedbackId].last_updated = new Date().toISOString();
      }

      return {
        success: true,
        message: 'Status updated successfully',
      };
    }

    const response = await apiClient.put<{ success: boolean; message: string }>(
      `/api/feedback/${feedbackId}/status`,
      request
    );
    return response.data;
  }

  /**
   * Generate AI response (stub for Phase 5)
   */
  async generateResponse(
    feedbackId: number
  ): Promise<{ success: boolean; message: string }> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Generate response:', feedbackId);
      throw new Error('Mock API: AI generation not implemented yet (Phase 5)');
    }

    const response = await apiClient.post<{ success: boolean; message: string }>(
      `/api/feedback/${feedbackId}/generate`
    );
    return response.data;
  }

  /**
   * Send response via email (stub for Phase 6)
   */
  async sendResponse(feedbackId: number): Promise<{ success: boolean; message: string }> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Send response:', feedbackId);
      throw new Error('Mock API: Email sending not implemented yet (Phase 6)');
    }

    const response = await apiClient.post<{ success: boolean; message: string }>(
      `/api/feedback/${feedbackId}/send`
    );
    return response.data;
  }
}

export const feedbackApi = new FeedbackApi();
export default feedbackApi;
```

---

### Task 4: Create Mock Feedback Data (`src/mockData/feedbackData.ts`)

**Purpose:** Mock data for development

**Create file:** `src/mockData/feedbackData.ts`

```typescript
/**
 * Mock feedback data for development
 */
import {
  Feedback,
  FeedbackListItem,
  FeedbackListResponse,
  FeedbackStatus,
  Sentiment,
} from '../types/feedback';

export const mockFeedbackList: FeedbackListResponse = {
  items: [
    {
      id: 1,
      venue_id: 'venue_001',
      venue_name: 'Downtown Bistro',
      customer_name: 'John Smith',
      rating: 5,
      comment_preview: 'Excellent service! The food was amazing and the staff was very friendly...',
      status: FeedbackStatus.SENT,
      sentiment: Sentiment.POSITIVE,
      created_at: '2025-01-15T14:30:00Z',
      has_response: true,
    },
    {
      id: 2,
      venue_id: 'venue_001',
      venue_name: 'Downtown Bistro',
      customer_name: 'Sarah Johnson',
      rating: 2,
      comment_preview: 'The wait time was too long and the food arrived cold. Very disappointed...',
      status: FeedbackStatus.GENERATED,
      sentiment: Sentiment.NEGATIVE,
      created_at: '2025-01-14T19:20:00Z',
      has_response: true,
    },
    {
      id: 3,
      venue_id: 'venue_002',
      venue_name: 'Sunset Cafe',
      customer_name: 'Michael Brown',
      rating: 4,
      comment_preview: 'Good experience overall. The atmosphere was nice, food was decent...',
      status: FeedbackStatus.PENDING,
      sentiment: Sentiment.POSITIVE,
      created_at: '2025-01-13T11:45:00Z',
      has_response: false,
    },
    // Add more mock items...
  ],
  total: 3,
  page: 1,
  page_size: 20,
  total_pages: 1,
};

export const mockFeedbackDetail: Record<number, Feedback> = {
  1: {
    id: 1,
    venue_id: 'venue_001',
    venue_name: 'Downtown Bistro',
    customer_email: 'john.smith@email.com',
    customer_name: 'John Smith',
    rating: 5,
    comment: 'Excellent service! The food was amazing and the staff was very friendly. Will definitely come back.',
    created_at: '2025-01-15T14:30:00Z',
    status: FeedbackStatus.SENT,
    sentiment: Sentiment.POSITIVE,
    ai_response_id: 'resp_123',
    last_updated: '2025-01-15T15:00:00Z',
    last_action_by: 'admin@encore.com',
    response_count: 1,
    email_sent_count: 1,
  },
  2: {
    id: 2,
    venue_id: 'venue_001',
    venue_name: 'Downtown Bistro',
    customer_email: 'sarah.j@email.com',
    customer_name: 'Sarah Johnson',
    rating: 2,
    comment: 'The wait time was too long and the food arrived cold. Very disappointed with the experience.',
    created_at: '2025-01-14T19:20:00Z',
    status: FeedbackStatus.GENERATED,
    sentiment: Sentiment.NEGATIVE,
    ai_response_id: 'resp_124',
    last_updated: '2025-01-14T19:45:00Z',
    last_action_by: 'admin@encore.com',
    response_count: 1,
    email_sent_count: 0,
  },
  3: {
    id: 3,
    venue_id: 'venue_002',
    venue_name: 'Sunset Cafe',
    customer_email: 'mbrown@email.com',
    customer_name: 'Michael Brown',
    rating: 4,
    comment: 'Good experience overall. The atmosphere was nice, food was decent. Could improve on presentation.',
    created_at: '2025-01-13T11:45:00Z',
    status: FeedbackStatus.PENDING,
    sentiment: Sentiment.POSITIVE,
    response_count: 0,
    email_sent_count: 0,
  },
};
```

---

### Task 5: Create Feedback List Hook (`src/hooks/useFeedbackList.ts`)

**Purpose:** Custom hook for feedback list management

**Create file:** `src/hooks/useFeedbackList.ts`

```typescript
/**
 * Custom hook for feedback list management
 */
import { useState, useEffect, useCallback } from 'react';
import { toast } from 'react-toastify';
import feedbackApi from '../api/feedbackApi';
import {
  FeedbackListItem,
  FeedbackFilter,
  FeedbackStatus,
  Sentiment,
} from '../types/feedback';

interface UseFeedbackListReturn {
  items: FeedbackListItem[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
  loading: boolean;
  error: string | null;
  filters: Partial<FeedbackFilter>;
  setFilters: (filters: Partial<FeedbackFilter>) => void;
  setPage: (page: number) => void;
  refresh: () => Promise<void>;
}

export const useFeedbackList = (
  initialFilters: Partial<FeedbackFilter> = {}
): UseFeedbackListReturn => {
  const [items, setItems] = useState<FeedbackListItem[]>([]);
  const [total, setTotal] = useState(0);
  const [page, setPageState] = useState(1);
  const [pageSize] = useState(20);
  const [totalPages, setTotalPages] = useState(1);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filters, setFiltersState] = useState<Partial<FeedbackFilter>>(initialFilters);

  const loadFeedback = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);

      const response = await feedbackApi.listFeedback({
        ...filters,
        page,
        page_size: pageSize,
      });

      setItems(response.items);
      setTotal(response.total);
      setTotalPages(response.total_pages);
    } catch (err: any) {
      const errorMessage = err.response?.data?.detail || err.message || 'Failed to load feedback';
      setError(errorMessage);
      toast.error(`Error loading feedback: ${errorMessage}`);
    } finally {
      setLoading(false);
    }
  }, [filters, page, pageSize]);

  const setFilters = useCallback((newFilters: Partial<FeedbackFilter>) => {
    setFiltersState(newFilters);
    setPageState(1); // Reset to page 1 when filters change
  }, []);

  const setPage = useCallback((newPage: number) => {
    setPageState(newPage);
  }, []);

  const refresh = useCallback(async () => {
    await loadFeedback();
  }, [loadFeedback]);

  // Load feedback when filters or page changes
  useEffect(() => {
    loadFeedback();
  }, [loadFeedback]);

  return {
    items,
    total,
    page,
    pageSize,
    totalPages,
    loading,
    error,
    filters,
    setFilters,
    setPage,
    refresh,
  };
};
```

---

### Task 6: Create Feedback Detail Hook (`src/hooks/useFeedbackDetail.ts`)

**Purpose:** Custom hook for single feedback management

**Create file:** `src/hooks/useFeedbackDetail.ts`

```typescript
/**
 * Custom hook for feedback detail management
 */
import { useState, useEffect, useCallback } from 'react';
import { toast } from 'react-toastify';
import feedbackApi from '../api/feedbackApi';
import { Feedback, FeedbackStatus } from '../types/feedback';

interface UseFeedbackDetailReturn {
  feedback: Feedback | null;
  loading: boolean;
  error: string | null;
  updateStatus: (status: FeedbackStatus, notes?: string) => Promise<boolean>;
  refresh: () => Promise<void>;
}

export const useFeedbackDetail = (feedbackId: number): UseFeedbackDetailReturn => {
  const [feedback, setFeedback] = useState<Feedback | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const loadFeedback = useCallback(async () => {
    if (!feedbackId) return;

    try {
      setLoading(true);
      setError(null);

      const data = await feedbackApi.getFeedback(feedbackId);
      setFeedback(data);
    } catch (err: any) {
      const errorMessage = err.response?.data?.detail || err.message || 'Failed to load feedback';
      setError(errorMessage);
      toast.error(`Error loading feedback: ${errorMessage}`);
    } finally {
      setLoading(false);
    }
  }, [feedbackId]);

  const updateStatus = useCallback(
    async (status: FeedbackStatus, notes?: string): Promise<boolean> => {
      if (!feedbackId) return false;

      try {
        setLoading(true);

        await feedbackApi.updateStatus(feedbackId, { status, notes });

        toast.success('Status updated successfully');

        // Reload feedback
        await loadFeedback();

        return true;
      } catch (err: any) {
        const errorMessage = err.response?.data?.detail || err.message || 'Failed to update status';
        toast.error(`Error: ${errorMessage}`);
        return false;
      } finally {
        setLoading(false);
      }
    },
    [feedbackId, loadFeedback]
  );

  const refresh = useCallback(async () => {
    await loadFeedback();
  }, [loadFeedback]);

  useEffect(() => {
    loadFeedback();
  }, [loadFeedback]);

  return {
    feedback,
    loading,
    error,
    updateStatus,
    refresh,
  };
};
```

---

### Task 7: Create UI Components

#### FeedbackStatusBadge Component

**Create file:** `src/components/FeedbackStatusBadge.tsx`

```typescript
/**
 * Status badge component
 */
import React from 'react';
import { FeedbackStatus, getStatusColor, getStatusLabel } from '../types/feedback';
import './FeedbackStatusBadge.css';

interface Props {
  status: FeedbackStatus;
}

export const FeedbackStatusBadge: React.FC<Props> = ({ status }) => {
  const color = getStatusColor(status);
  const label = getStatusLabel(status);

  return (
    <span
      className="feedback-status-badge"
      style={{ backgroundColor: color }}
    >
      {label}
    </span>
  );
};
```

**Create file:** `src/components/FeedbackStatusBadge.css`

```css
.feedback-status-badge {
  display: inline-block;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
  color: white;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}
```

#### Pagination Component

**Create file:** `src/components/Pagination.tsx`

```typescript
/**
 * Pagination component
 */
import React from 'react';
import './Pagination.css';

interface Props {
  currentPage: number;
  totalPages: number;
  onPageChange: (page: number) => void;
}

export const Pagination: React.FC<Props> = ({
  currentPage,
  totalPages,
  onPageChange,
}) => {
  if (totalPages <= 1) return null;

  const pages: number[] = [];
  const maxVisible = 5;

  let start = Math.max(1, currentPage - Math.floor(maxVisible / 2));
  let end = Math.min(totalPages, start + maxVisible - 1);

  if (end - start + 1 < maxVisible) {
    start = Math.max(1, end - maxVisible + 1);
  }

  for (let i = start; i <= end; i++) {
    pages.push(i);
  }

  return (
    <div className="pagination">
      <button
        onClick={() => onPageChange(currentPage - 1)}
        disabled={currentPage === 1}
        className="pagination-button"
      >
        Previous
      </button>

      {start > 1 && (
        <>
          <button onClick={() => onPageChange(1)} className="pagination-button">
            1
          </button>
          {start > 2 && <span className="pagination-ellipsis">...</span>}
        </>
      )}

      {pages.map((page) => (
        <button
          key={page}
          onClick={() => onPageChange(page)}
          className={`pagination-button ${page === currentPage ? 'active' : ''}`}
        >
          {page}
        </button>
      ))}

      {end < totalPages && (
        <>
          {end < totalPages - 1 && <span className="pagination-ellipsis">...</span>}
          <button onClick={() => onPageChange(totalPages)} className="pagination-button">
            {totalPages}
          </button>
        </>
      )}

      <button
        onClick={() => onPageChange(currentPage + 1)}
        disabled={currentPage === totalPages}
        className="pagination-button"
      >
        Next
      </button>
    </div>
  );
};
```

**Create file:** `src/components/Pagination.css`

```css
.pagination {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 20px;
  justify-content: center;
}

.pagination-button {
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  background: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
}

.pagination-button:hover:not(:disabled) {
  background: #f3f4f6;
  border-color: #9ca3af;
}

.pagination-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.pagination-button.active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.pagination-ellipsis {
  padding: 8px 4px;
  color: #6b7280;
}
```

---

### Task 8: Update Feedback List Page

**Update file:** `src/pages/FeedbackList.tsx`

Replace mock data usage with the real API hook. Key changes:
1. Import `useFeedbackList` hook
2. Use hook instead of local state
3. Add filtering UI
4. Add pagination component
5. Keep mock toggle working

---

### Task 9: Update Feedback Detail Page

**Update file:** `src/pages/FeedbackDetail.tsx`

Replace mock data with real API. Key changes:
1. Import `useFeedbackDetail` hook
2. Use hook for data and status updates
3. Add loading/error states
4. Show status update UI

---

## 🔍 Verification Checklist

- [ ] All TypeScript files compile without errors
- [ ] Feedback list loads in mock mode
- [ ] Feedback list loads in real API mode
- [ ] Filtering works (status, sentiment, rating, search)
- [ ] Pagination works correctly
- [ ] Can view feedback details
- [ ] Can update feedback status
- [ ] Loading states show correctly
- [ ] Error handling works
- [ ] Toast notifications appear

---

## 📝 Testing Steps

1. **Mock Mode:**
   ```bash
   # .env.local
   REACT_APP_USE_MOCK_API=true
   npm start
   ```
   - Navigate to Feedback List
   - Test filters
   - Test pagination
   - Click on feedback item
   - View details
   - Update status

2. **Real API Mode:**
   ```bash
   # .env.local
   REACT_APP_USE_MOCK_API=false

   # Start backend first
   cd encore_backend && python main.py

   # Then frontend
   npm start
   ```
   - Same tests as mock mode
   - Verify data comes from backend

---

## ✅ Completion Criteria

- [ ] Feedback types defined
- [ ] Feedback API service created
- [ ] Mock data available
- [ ] List hook implemented
- [ ] Detail hook implemented
- [ ] Status badge component created
- [ ] Pagination component created
- [ ] Feedback List page updated
- [ ] Feedback Detail page updated
- [ ] End-to-end testing complete

---

## 🔄 Handoff to Phase 5

**Next steps:**
- Implement AI response generation (Phase 5A backend)
- Integrate AI features in frontend (Phase 5B)
- Add response editing capabilities
- Test AI generation flow
