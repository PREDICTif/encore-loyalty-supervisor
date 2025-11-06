# Phase 3B: Frontend Settings Integration

**Assigned to:** AI Worker (Frontend Developer)
**Model:** Claude Sonnet 4.5
**Environment:** encore_frontend directory
**Prerequisites:**
- Phase 1 completed (Foundation setup)
- Phase 2B completed (Authentication working)
- Phase 3A completed (Backend settings API working)

## 📋 Overview

Integrate the **Settings Management** system into the React frontend. This includes creating the Settings page UI, connecting to real backend API, and maintaining the mock API toggle for development.

## 🎯 Objectives

1. Create settings API service layer
2. Update Settings page to use real API
3. Add settings state management
4. Implement settings forms with validation
5. Add loading and error states
6. Maintain mock API toggle functionality
7. Test complete settings flow

## 📁 Directory Structure

```
encore_frontend/
├── src/
│   ├── api/
│   │   └── settingsApi.ts          # ← CREATE
│   ├── types/
│   │   └── settings.ts              # ← CREATE
│   ├── pages/
│   │   └── Settings.tsx             # ← UPDATE (connect to real API)
│   ├── hooks/
│   │   └── useSettings.ts           # ← CREATE (custom hook)
│   └── mockData/
│       └── settingsData.ts          # ← KEEP (for mock mode)
```

## ✅ Task Checklist

### Task 1: Create Settings Types (`src/types/settings.ts`)

**Purpose:** TypeScript interfaces matching backend models

**Create file:** `src/types/settings.ts`

```typescript
/**
 * Settings type definitions matching backend models
 */

export interface AISettings {
  enabled: boolean;
  model_version: string;
  temperature: number; // 0.0 to 1.0
  max_tokens: number; // 100 to 2000
  system_prompt?: string;
  auto_respond: boolean;
}

export interface EmailSettings {
  enabled: boolean;
  from_email?: string;
  reply_to?: string;
  signature?: string;
  auto_send: boolean;
}

export interface NotificationSettings {
  email_on_feedback: boolean;
  email_on_response: boolean;
  slack_webhook?: string;
}

export interface VenueSettings {
  venue_id: string;
  venue_name: string;
  ai_settings: AISettings;
  email_settings: EmailSettings;
  notification_settings: NotificationSettings;
  custom_fields: Record<string, any>;
  created_at?: string;
  updated_at?: string;
  updated_by?: string;
}

export interface UpdateVenueSettingsRequest {
  ai_settings?: Partial<AISettings>;
  email_settings?: Partial<EmailSettings>;
  notification_settings?: Partial<NotificationSettings>;
  custom_fields?: Record<string, any>;
}

export interface VenueSettingsResponse {
  success: boolean;
  message: string;
  settings?: VenueSettings;
}

// Default settings factory
export const createDefaultSettings = (venueId: string, venueName: string): VenueSettings => ({
  venue_id: venueId,
  venue_name: venueName,
  ai_settings: {
    enabled: false,
    model_version: 'claude-3-5-sonnet-20241022',
    temperature: 0.7,
    max_tokens: 500,
    auto_respond: false,
  },
  email_settings: {
    enabled: false,
    auto_send: false,
  },
  notification_settings: {
    email_on_feedback: true,
    email_on_response: false,
  },
  custom_fields: {},
});
```

---

### Task 2: Create Settings API Service (`src/api/settingsApi.ts`)

**Purpose:** API calls to backend settings endpoints

**Create file:** `src/api/settingsApi.ts`

```typescript
/**
 * Settings API service
 * Handles all settings-related API calls
 */
import apiClient from './apiClient';
import {
  VenueSettings,
  UpdateVenueSettingsRequest,
  VenueSettingsResponse,
} from '../types/settings';
import { mockSettings } from '../mockData/settingsData';
import { API_CONFIG } from '../config/api';

class SettingsApi {
  /**
   * Get settings for a specific venue
   */
  async getSettings(venueId: string): Promise<VenueSettings> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Getting settings for venue:', venueId);
      // Simulate API delay
      await new Promise((resolve) => setTimeout(resolve, 500));
      return mockSettings[venueId] || mockSettings['default'];
    }

    const response = await apiClient.get<VenueSettings>(
      `/api/settings/${venueId}`
    );
    return response.data;
  }

  /**
   * Update settings for a specific venue
   */
  async updateSettings(
    venueId: string,
    updates: UpdateVenueSettingsRequest
  ): Promise<VenueSettingsResponse> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Updating settings for venue:', venueId, updates);
      // Simulate API delay
      await new Promise((resolve) => setTimeout(resolve, 800));

      // Update mock data
      if (mockSettings[venueId]) {
        if (updates.ai_settings) {
          mockSettings[venueId].ai_settings = {
            ...mockSettings[venueId].ai_settings,
            ...updates.ai_settings,
          };
        }
        if (updates.email_settings) {
          mockSettings[venueId].email_settings = {
            ...mockSettings[venueId].email_settings,
            ...updates.email_settings,
          };
        }
        if (updates.notification_settings) {
          mockSettings[venueId].notification_settings = {
            ...mockSettings[venueId].notification_settings,
            ...updates.notification_settings,
          };
        }
        mockSettings[venueId].updated_at = new Date().toISOString();
      }

      return {
        success: true,
        message: 'Settings updated successfully',
        settings: mockSettings[venueId] || mockSettings['default'],
      };
    }

    const response = await apiClient.put<VenueSettingsResponse>(
      `/api/settings/${venueId}`,
      updates
    );
    return response.data;
  }

  /**
   * Create settings for a new venue (admin only)
   */
  async createSettings(
    venueId: string,
    venueName: string
  ): Promise<VenueSettingsResponse> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Creating settings for venue:', venueId);
      throw new Error('Mock API: Create settings not implemented');
    }

    const response = await apiClient.post<VenueSettingsResponse>(
      `/api/settings/${venueId}`,
      { venue_name: venueName }
    );
    return response.data;
  }

  /**
   * Delete settings for a venue (admin only, use with caution)
   */
  async deleteSettings(venueId: string): Promise<{ success: boolean; message: string }> {
    // Check if using mock API
    if (API_CONFIG.USE_MOCK_API) {
      console.log('[MOCK] Deleting settings for venue:', venueId);
      throw new Error('Mock API: Delete settings not implemented');
    }

    const response = await apiClient.delete<{ success: boolean; message: string }>(
      `/api/settings/${venueId}`
    );
    return response.data;
  }
}

export const settingsApi = new SettingsApi();
export default settingsApi;
```

---

### Task 3: Create Mock Settings Data (`src/mockData/settingsData.ts`)

**Purpose:** Mock data for development without backend

**Create file:** `src/mockData/settingsData.ts`

```typescript
/**
 * Mock settings data for development
 */
import { VenueSettings, createDefaultSettings } from '../types/settings';

export const mockSettings: Record<string, VenueSettings> = {
  'venue_001': {
    venue_id: 'venue_001',
    venue_name: 'Downtown Bistro',
    ai_settings: {
      enabled: true,
      model_version: 'claude-3-5-sonnet-20241022',
      temperature: 0.8,
      max_tokens: 750,
      system_prompt: 'You are a helpful restaurant assistant.',
      auto_respond: true,
    },
    email_settings: {
      enabled: true,
      from_email: 'noreply@downtownbistro.com',
      reply_to: 'feedback@downtownbistro.com',
      signature: 'Thanks for your feedback!\\n\\nThe Downtown Bistro Team',
      auto_send: false,
    },
    notification_settings: {
      email_on_feedback: true,
      email_on_response: true,
      slack_webhook: 'https://hooks.slack.com/services/XXXXX',
    },
    custom_fields: {
      business_hours: '9am-10pm',
      timezone: 'America/New_York',
    },
    created_at: '2025-01-01T00:00:00Z',
    updated_at: '2025-01-15T10:30:00Z',
    updated_by: 'admin@encore.com',
  },
  'venue_002': {
    venue_id: 'venue_002',
    venue_name: 'Sunset Cafe',
    ai_settings: {
      enabled: false,
      model_version: 'claude-3-5-sonnet-20241022',
      temperature: 0.7,
      max_tokens: 500,
      auto_respond: false,
    },
    email_settings: {
      enabled: false,
      auto_send: false,
    },
    notification_settings: {
      email_on_feedback: true,
      email_on_response: false,
    },
    custom_fields: {},
    created_at: '2025-01-05T00:00:00Z',
    updated_at: '2025-01-05T00:00:00Z',
    updated_by: 'admin@encore.com',
  },
  // Default fallback
  'default': createDefaultSettings('default', 'Default Venue'),
};
```

---

### Task 4: Create Settings Hook (`src/hooks/useSettings.ts`)

**Purpose:** Custom React hook for settings state management

**Create file:** `src/hooks/useSettings.ts`

```typescript
/**
 * Custom hook for settings management
 */
import { useState, useEffect, useCallback } from 'react';
import { toast } from 'react-toastify';
import settingsApi from '../api/settingsApi';
import {
  VenueSettings,
  UpdateVenueSettingsRequest,
} from '../types/settings';

interface UseSettingsReturn {
  settings: VenueSettings | null;
  loading: boolean;
  error: string | null;
  updateSettings: (updates: UpdateVenueSettingsRequest) => Promise<boolean>;
  refreshSettings: () => Promise<void>;
}

export const useSettings = (venueId: string): UseSettingsReturn => {
  const [settings, setSettings] = useState<VenueSettings | null>(null);
  const [loading, setLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);

  /**
   * Load settings from API
   */
  const loadSettings = useCallback(async () => {
    if (!venueId) {
      setError('No venue ID provided');
      setLoading(false);
      return;
    }

    try {
      setLoading(true);
      setError(null);

      const data = await settingsApi.getSettings(venueId);
      setSettings(data);
    } catch (err: any) {
      const errorMessage = err.response?.data?.detail || err.message || 'Failed to load settings';
      setError(errorMessage);
      toast.error(`Error loading settings: ${errorMessage}`);
    } finally {
      setLoading(false);
    }
  }, [venueId]);

  /**
   * Update settings
   */
  const updateSettings = useCallback(
    async (updates: UpdateVenueSettingsRequest): Promise<boolean> => {
      if (!venueId) {
        toast.error('No venue ID provided');
        return false;
      }

      try {
        setLoading(true);
        setError(null);

        const response = await settingsApi.updateSettings(venueId, updates);

        if (response.success && response.settings) {
          setSettings(response.settings);
          toast.success(response.message || 'Settings updated successfully');
          return true;
        } else {
          toast.error(response.message || 'Failed to update settings');
          return false;
        }
      } catch (err: any) {
        const errorMessage = err.response?.data?.detail || err.message || 'Failed to update settings';
        setError(errorMessage);
        toast.error(`Error updating settings: ${errorMessage}`);
        return false;
      } finally {
        setLoading(false);
      }
    },
    [venueId]
  );

  /**
   * Refresh settings (useful after external updates)
   */
  const refreshSettings = useCallback(async () => {
    await loadSettings();
  }, [loadSettings]);

  // Load settings on mount or when venueId changes
  useEffect(() => {
    loadSettings();
  }, [loadSettings]);

  return {
    settings,
    loading,
    error,
    updateSettings,
    refreshSettings,
  };
};
```

---

### Task 5: Update Settings Page (`src/pages/Settings.tsx`)

**Purpose:** Connect UI to real API

**Update file:** `src/pages/Settings.tsx`

**IMPORTANT:** This is a significant update. Replace the mock data usage with the real API hook.

**Key changes:**
1. Import `useSettings` hook instead of mock data
2. Use `settings` from hook instead of local state
3. Call `updateSettings` function instead of local state update
4. Add loading and error states to UI
5. Keep all existing UI components and styling

**Example structure:**

```typescript
import React, { useState } from 'react';
import { useSettings } from '../hooks/useSettings';
import { UpdateVenueSettingsRequest } from '../types/settings';
import './Settings.css';

const Settings: React.FC = () => {
  // Get current venue from auth context (or use first venue for now)
  const currentVenueId = 'venue_001'; // TODO: Get from auth context

  // Use settings hook
  const { settings, loading, error, updateSettings } = useSettings(currentVenueId);

  // Local form state
  const [isSaving, setIsSaving] = useState(false);

  // Handle AI settings update
  const handleAISettingsChange = async (field: string, value: any) => {
    if (!settings) return;

    setIsSaving(true);

    const updates: UpdateVenueSettingsRequest = {
      ai_settings: {
        ...settings.ai_settings,
        [field]: value,
      },
    };

    await updateSettings(updates);
    setIsSaving(false);
  };

  // Handle email settings update
  const handleEmailSettingsChange = async (field: string, value: any) => {
    if (!settings) return;

    setIsSaving(true);

    const updates: UpdateVenueSettingsRequest = {
      email_settings: {
        ...settings.email_settings,
        [field]: value,
      },
    };

    await updateSettings(updates);
    setIsSaving(false);
  };

  // Loading state
  if (loading && !settings) {
    return (
      <div className="settings-page">
        <div className="loading-spinner">Loading settings...</div>
      </div>
    );
  }

  // Error state
  if (error && !settings) {
    return (
      <div className="settings-page">
        <div className="error-message">
          <h3>Error Loading Settings</h3>
          <p>{error}</p>
        </div>
      </div>
    );
  }

  // No settings state
  if (!settings) {
    return (
      <div className="settings-page">
        <div className="error-message">No settings available</div>
      </div>
    );
  }

  return (
    <div className="settings-page">
      <h1>Settings - {settings.venue_name}</h1>

      {/* AI Settings Section */}
      <section className="settings-section">
        <h2>AI Settings</h2>

        <div className="setting-item">
          <label>
            <input
              type="checkbox"
              checked={settings.ai_settings.enabled}
              onChange={(e) => handleAISettingsChange('enabled', e.target.checked)}
              disabled={isSaving}
            />
            Enable AI Responses
          </label>
        </div>

        <div className="setting-item">
          <label>Temperature ({settings.ai_settings.temperature})</label>
          <input
            type="range"
            min="0"
            max="1"
            step="0.1"
            value={settings.ai_settings.temperature}
            onChange={(e) => handleAISettingsChange('temperature', parseFloat(e.target.value))}
            disabled={isSaving || !settings.ai_settings.enabled}
          />
        </div>

        <div className="setting-item">
          <label>Max Tokens</label>
          <input
            type="number"
            min="100"
            max="2000"
            value={settings.ai_settings.max_tokens}
            onChange={(e) => handleAISettingsChange('max_tokens', parseInt(e.target.value))}
            disabled={isSaving || !settings.ai_settings.enabled}
          />
        </div>

        <div className="setting-item">
          <label>
            <input
              type="checkbox"
              checked={settings.ai_settings.auto_respond}
              onChange={(e) => handleAISettingsChange('auto_respond', e.target.checked)}
              disabled={isSaving || !settings.ai_settings.enabled}
            />
            Auto-respond to feedback
          </label>
        </div>
      </section>

      {/* Email Settings Section */}
      <section className="settings-section">
        <h2>Email Settings</h2>

        <div className="setting-item">
          <label>
            <input
              type="checkbox"
              checked={settings.email_settings.enabled}
              onChange={(e) => handleEmailSettingsChange('enabled', e.target.checked)}
              disabled={isSaving}
            />
            Enable Email Sending
          </label>
        </div>

        <div className="setting-item">
          <label>From Email</label>
          <input
            type="email"
            value={settings.email_settings.from_email || ''}
            onChange={(e) => handleEmailSettingsChange('from_email', e.target.value)}
            disabled={isSaving || !settings.email_settings.enabled}
            placeholder="noreply@yourrestaurant.com"
          />
        </div>

        <div className="setting-item">
          <label>Reply-To Email</label>
          <input
            type="email"
            value={settings.email_settings.reply_to || ''}
            onChange={(e) => handleEmailSettingsChange('reply_to', e.target.value)}
            disabled={isSaving || !settings.email_settings.enabled}
            placeholder="feedback@yourrestaurant.com"
          />
        </div>

        <div className="setting-item">
          <label>Email Signature</label>
          <textarea
            value={settings.email_settings.signature || ''}
            onChange={(e) => handleEmailSettingsChange('signature', e.target.value)}
            disabled={isSaving || !settings.email_settings.enabled}
            placeholder="Thanks,\\nYour Team"
            rows={3}
          />
        </div>

        <div className="setting-item">
          <label>
            <input
              type="checkbox"
              checked={settings.email_settings.auto_send}
              onChange={(e) => handleEmailSettingsChange('auto_send', e.target.checked)}
              disabled={isSaving || !settings.email_settings.enabled}
            />
            Auto-send email responses
          </label>
        </div>
      </section>

      {/* Saving indicator */}
      {isSaving && (
        <div className="saving-indicator">Saving changes...</div>
      )}

      {/* Metadata footer */}
      <div className="settings-metadata">
        <small>
          Last updated: {settings.updated_at ? new Date(settings.updated_at).toLocaleString() : 'Never'}
          {settings.updated_by && ` by ${settings.updated_by}`}
        </small>
      </div>
    </div>
  );
};

export default Settings;
```

**Note:** Adapt this to match your existing Settings.tsx structure and styling. The key changes are:
- Replace mock data with `useSettings` hook
- Add loading/error states
- Call `updateSettings` instead of local state updates
- Add `isSaving` state for better UX

---

### Task 6: Add Settings to API Client Exports

**Purpose:** Make settingsApi easily importable

**Update file:** `src/api/index.ts` (create if doesn't exist)

```typescript
/**
 * Centralized API exports
 */
export { default as authApi } from './authApi';
export { default as settingsApi } from './settingsApi';
// Add more API exports as you create them
```

---

### Task 7: Test Settings Integration

**Purpose:** Verify complete flow works

**Manual Testing Steps:**

1. **Test Mock Mode:**
   ```bash
   # In .env.local
   REACT_APP_USE_MOCK_API=true

   # Start frontend
   npm start

   # Navigate to Settings page
   # ✅ Should load mock settings
   # ✅ Should update settings locally
   # ✅ Should show "MOCK" in console logs
   ```

2. **Test Real API Mode:**
   ```bash
   # In .env.local
   REACT_APP_USE_MOCK_API=false

   # Make sure backend is running
   cd /home/ec2-user/encore-loyalty-new-system/encore_backend
   source ../venv/bin/activate
   python main.py

   # Start frontend
   npm start

   # Navigate to Settings page
   # ✅ Should load settings from backend
   # ✅ Should update settings via API
   # ✅ Should show success toasts
   # ✅ Should handle errors gracefully
   ```

3. **Test Error Handling:**
   - Stop backend while frontend is running
   - Try to load settings → should show error message
   - Try to update settings → should show error toast

4. **Test Authentication:**
   - Logout and try to access Settings → should redirect to login
   - Login and access Settings → should work

---

## 🔍 Verification Checklist

After completing all tasks:

- [ ] `src/types/settings.ts` created with all interfaces
- [ ] `src/api/settingsApi.ts` created with all API methods
- [ ] `src/mockData/settingsData.ts` created with sample data
- [ ] `src/hooks/useSettings.ts` created and working
- [ ] `src/pages/Settings.tsx` updated to use real API
- [ ] No TypeScript errors in any file
- [ ] Frontend compiles successfully (`npm start`)
- [ ] Settings page loads in mock mode
- [ ] Settings page loads in real API mode (with backend running)
- [ ] Can update AI settings via UI
- [ ] Can update Email settings via UI
- [ ] Loading states show correctly
- [ ] Error states show correctly
- [ ] Success toasts appear on successful updates
- [ ] Error toasts appear on failed updates

---

## 📝 Testing Commands

```bash
# 1. Check TypeScript compilation
cd /home/ec2-user/encore-loyalty-new-system/encore_frontend
npm run build

# 2. Start in mock mode
# Edit .env.local: REACT_APP_USE_MOCK_API=true
npm start

# 3. Start in real API mode
# Edit .env.local: REACT_APP_USE_MOCK_API=false
# Make sure backend is running first!
npm start

# 4. Run in production mode
npm run build
serve -s build
```

---

## 🚨 Important Notes

1. **Mock Mode Toggle:** Always respect `REACT_APP_USE_MOCK_API` flag
2. **Error Handling:** All API calls wrapped in try/catch with user feedback
3. **Loading States:** Show loading indicators for better UX
4. **Validation:** Rely on backend validation, but add frontend validation for better UX
5. **Toast Notifications:** Use react-toastify for success/error messages
6. **Type Safety:** All API responses typed with TypeScript interfaces

---

## 🎨 UI/UX Considerations

1. **Loading States:**
   - Show spinner while loading settings
   - Disable form inputs while saving
   - Show "Saving..." indicator

2. **Error States:**
   - Display friendly error messages
   - Provide retry mechanisms
   - Don't lose user input on errors

3. **Success Feedback:**
   - Show toast notification on successful save
   - Update metadata (last updated time)
   - Auto-save vs manual save (choose one)

4. **Disabled States:**
   - Disable child settings when parent is disabled (e.g., AI settings when AI is disabled)
   - Show visual indication of disabled state

---

## ✅ Completion Criteria

- [ ] All TypeScript files created without errors
- [ ] Settings page successfully loads settings from API
- [ ] Settings page successfully updates settings via API
- [ ] Mock mode works correctly for development
- [ ] Real API mode works correctly with backend
- [ ] Loading and error states work properly
- [ ] Toast notifications appear for success/error
- [ ] No console errors in browser
- [ ] Responsive design maintained

---

## 🔄 Handoff to Phase 4

Once frontend settings integration is complete:

**Status to report:**
- ✅ Settings types defined
- ✅ Settings API service created
- ✅ Settings hook implemented
- ✅ Settings page updated with real API
- ✅ Mock mode toggle working
- ✅ End-to-end testing complete

**Next steps (Phase 4):**
- Implement Feedback system backend (Phase 4A)
- Integrate Feedback system frontend (Phase 4B)
- Connect feedback with settings (AI auto-respond, email auto-send)
