# Phase 5B: Frontend AI Response Features

## 🎯 OBJECTIVE
Integrate AI response generation capabilities into the frontend, providing intuitive UI for generating, reviewing, editing, and managing AI-powered feedback responses.

## 📋 CONTEXT

### Current State (Phase 5A Complete)
- ✅ Backend AI endpoints implemented
- ✅ DynamoDB storage for responses
- ✅ Prompt engineering configured
- ✅ Response versioning working
- ✅ Frontend feedback system operational

### What Already Exists
1. **Mockup Reference** (`encore_archive/mockup-backup/`):
   - AI generation UI mockups
   - Response review interface
   - Loading states and animations

2. **Frontend Infrastructure**:
   - Feedback list and detail pages
   - API client with interceptors
   - Mock/Real API toggle
   - React Query for data fetching

3. **Backend APIs** (from Phase 5A):
   - `POST /api/ai/generate-response/{feedback_id}`
   - `POST /api/ai/regenerate/{feedback_id}`
   - `GET /api/ai/response/{feedback_id}`

### What Needs to Be Built
1. **AI Service Layer**: API integration for AI endpoints
2. **UI Components**: Generation, review, edit interfaces
3. **State Management**: Response versioning and history
4. **Loading States**: Professional generation animations
5. **Error Handling**: User-friendly error messages

## 🏗️ ARCHITECTURE

### Component Structure
```
src/
├── services/
│   └── api/
│       └── aiApi.ts          # AI response API service
├── hooks/
│   ├── useAIResponse.ts      # AI response data hook
│   └── useAIGeneration.ts   # Generation action hook
├── components/
│   └── ai/
│       ├── AIResponseGenerator.tsx
│       ├── AIResponseViewer.tsx
│       ├── AIResponseEditor.tsx
│       ├── RegenerationModal.tsx
│       └── ResponseHistory.tsx
└── types/
    └── index.ts              # AI response types
```

## 📝 IMPLEMENTATION TASKS

### Task 1: Create TypeScript Types (30 min)

Update `src/types/index.ts`:

```typescript
// AI Response Types
export interface AIResponseGenerationRequest {
  customInstructions?: string;
  temperature?: number;
  regenerate?: boolean;
}

export interface SentimentAnalysis {
  sentiment: 'positive' | 'neutral' | 'negative';
  satisfactionScore: number;
  keyIssues: string[];
}

export interface GenerationMetadata {
  modelId: string;
  promptVersion: string;
  temperature: number;
  customInstructions?: string;
  generationTimeMs: number;
}

export interface AIResponse {
  feedbackId: string;
  version: string;
  responseText: string;
  sentimentAnalysis: SentimentAnalysis;
  generationMetadata: GenerationMetadata;
  status: 'draft' | 'approved' | 'sent';
  createdAt: string;
  createdBy: string;
  approvedAt?: string;
  approvedBy?: string;
  sentAt?: string;
}

export interface AIResponseHistory {
  feedbackId: string;
  currentVersion: AIResponse;
  history?: AIResponseVersion[];
}

export interface AIResponseVersion {
  version: string;
  createdAt: string;
  status: string;
  createdBy: string;
}

// Generation State
export interface AIGenerationState {
  isGenerating: boolean;
  progress: number;
  currentStep: string;
  error?: string;
}
```

### Task 2: Create AI API Service (1 hour)

Create `src/services/api/aiApi.ts`:

```typescript
import { apiClient } from '../apiClient';
import { API_CONFIG } from '../../config/api.config';
import {
  AIResponse,
  AIResponseHistory,
  AIResponseGenerationRequest
} from '../../types';

class AIApiService {
  /**
   * Generate AI response for feedback
   */
  async generateResponse(
    feedbackId: string,
    request?: AIResponseGenerationRequest
  ): Promise<AIResponse> {
    if (API_CONFIG.USE_MOCK_API) {
      // Return mock response
      return this.mockGenerateResponse(feedbackId, request);
    }

    const response = await apiClient.post(
      `/api/ai/generate-response/${feedbackId}`,
      request || {}
    );
    return response.data;
  }

  /**
   * Regenerate response with custom instructions
   */
  async regenerateResponse(
    feedbackId: string,
    customInstructions: string,
    previousVersion?: string
  ): Promise<AIResponse> {
    if (API_CONFIG.USE_MOCK_API) {
      return this.mockGenerateResponse(feedbackId, {
        customInstructions,
        regenerate: true
      });
    }

    const response = await apiClient.post(
      `/api/ai/regenerate/${feedbackId}`,
      {
        customInstructions,
        previousVersion
      }
    );
    return response.data;
  }

  /**
   * Get AI response for feedback
   */
  async getResponse(
    feedbackId: string,
    version?: string,
    includeHistory: boolean = false
  ): Promise<AIResponseHistory> {
    if (API_CONFIG.USE_MOCK_API) {
      return this.mockGetResponse(feedbackId, includeHistory);
    }

    const params = new URLSearchParams();
    if (version) params.append('version', version);
    if (includeHistory) params.append('include_history', 'true');

    const response = await apiClient.get(
      `/api/ai/response/${feedbackId}?${params.toString()}`
    );
    return response.data;
  }

  // Mock implementations
  private async mockGenerateResponse(
    feedbackId: string,
    request?: AIResponseGenerationRequest
  ): Promise<AIResponse> {
    // Simulate API delay
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    return {
      feedbackId,
      version: `v1_${new Date().toISOString()}`,
      responseText: `Dear Valued Customer,\n\nThank you for taking the time to share your feedback...`,
      sentimentAnalysis: {
        sentiment: 'negative',
        satisfactionScore: 0.3,
        keyIssues: ['service', 'wait time']
      },
      generationMetadata: {
        modelId: 'claude-3.5-sonnet',
        promptVersion: 'v1.0',
        temperature: request?.temperature || 0.7,
        customInstructions: request?.customInstructions,
        generationTimeMs: 2000
      },
      status: 'draft',
      createdAt: new Date().toISOString(),
      createdBy: 'current-user-id'
    };
  }

  private async mockGetResponse(
    feedbackId: string,
    includeHistory: boolean
  ): Promise<AIResponseHistory> {
    await new Promise(resolve => setTimeout(resolve, 500));
    
    const currentVersion = await this.mockGenerateResponse(feedbackId);
    
    return {
      feedbackId,
      currentVersion,
      history: includeHistory ? [
        {
          version: 'v0_2025-11-19T09:00:00Z',
          createdAt: '2025-11-19T09:00:00Z',
          status: 'superseded',
          createdBy: 'user-123'
        }
      ] : undefined
    };
  }
}

export const aiApi = new AIApiService();
```

### Task 3: Create React Hooks (1 hour)

Create `src/hooks/useAIResponse.ts`:

```typescript
import { useQuery, useMutation, useQueryClient } from 'react-query';
import { aiApi } from '../services/api/aiApi';
import { toast } from 'react-toastify';

export const useAIResponse = (feedbackId: string) => {
  const queryClient = useQueryClient();

  // Query for fetching AI response
  const responseQuery = useQuery(
    ['ai-response', feedbackId],
    () => aiApi.getResponse(feedbackId, undefined, true),
    {
      enabled: !!feedbackId,
      staleTime: 30000, // 30 seconds
    }
  );

  // Mutation for generating response
  const generateMutation = useMutation(
    (customInstructions?: string) => 
      aiApi.generateResponse(feedbackId, { customInstructions }),
    {
      onSuccess: (data) => {
        queryClient.invalidateQueries(['ai-response', feedbackId]);
        toast.success('AI response generated successfully!');
      },
      onError: (error: any) => {
        toast.error(error.response?.data?.detail || 'Failed to generate response');
      }
    }
  );

  // Mutation for regenerating response
  const regenerateMutation = useMutation(
    ({ customInstructions, previousVersion }: {
      customInstructions: string;
      previousVersion?: string;
    }) => aiApi.regenerateResponse(feedbackId, customInstructions, previousVersion),
    {
      onSuccess: (data) => {
        queryClient.invalidateQueries(['ai-response', feedbackId]);
        toast.success('Response regenerated successfully!');
      },
      onError: (error: any) => {
        toast.error(error.response?.data?.detail || 'Failed to regenerate response');
      }
    }
  );

  return {
    // Data
    response: responseQuery.data?.currentVersion,
    history: responseQuery.data?.history,
    isLoading: responseQuery.isLoading,
    error: responseQuery.error,
    
    // Actions
    generateResponse: generateMutation.mutate,
    regenerateResponse: regenerateMutation.mutate,
    isGenerating: generateMutation.isLoading || regenerateMutation.isLoading,
    
    // Utilities
    refetch: responseQuery.refetch,
    hasResponse: !!responseQuery.data?.currentVersion
  };
};
```

### Task 4: Create AI Response Components (2-3 hours)

#### 4.1 AIResponseGenerator Component

Create `src/components/ai/AIResponseGenerator.tsx`:

```typescript
import React, { useState } from 'react';
import { AIGenerationState } from '../../types';

interface AIResponseGeneratorProps {
  feedbackId: string;
  onGenerate: (customInstructions?: string) => void;
  isGenerating: boolean;
  hasExistingResponse: boolean;
}

export const AIResponseGenerator: React.FC<AIResponseGeneratorProps> = ({
  feedbackId,
  onGenerate,
  isGenerating,
  hasExistingResponse
}) => {
  const [showCustomInstructions, setShowCustomInstructions] = useState(false);
  const [customInstructions, setCustomInstructions] = useState('');
  const [generationState, setGenerationState] = useState<AIGenerationState>({
    isGenerating: false,
    progress: 0,
    currentStep: ''
  });

  // Simulate generation progress
  const handleGenerate = () => {
    onGenerate(customInstructions || undefined);
    
    if (isGenerating) {
      // Simulate progress updates
      setGenerationState({
        isGenerating: true,
        progress: 0,
        currentStep: 'Analyzing feedback...'
      });

      const steps = [
        { progress: 25, step: 'Understanding context...' },
        { progress: 50, step: 'Crafting personalized response...' },
        { progress: 75, step: 'Reviewing tone and content...' },
        { progress: 100, step: 'Finalizing response...' }
      ];

      steps.forEach((step, index) => {
        setTimeout(() => {
          setGenerationState({
            isGenerating: true,
            progress: step.progress,
            currentStep: step.step
          });
        }, (index + 1) * 500);
      });

      setTimeout(() => {
        setGenerationState({
          isGenerating: false,
          progress: 100,
          currentStep: 'Complete!'
        });
      }, 2500);
    }
  };

  return (
    <div className="bg-white rounded-lg shadow p-6">
      <h3 className="text-lg font-semibold mb-4">
        AI Response Generation
      </h3>

      {!hasExistingResponse ? (
        <div className="space-y-4">
          <p className="text-gray-600">
            No AI response has been generated for this feedback yet.
          </p>

          <button
            onClick={() => setShowCustomInstructions(!showCustomInstructions)}
            className="text-blue-600 hover:text-blue-700 text-sm"
          >
            {showCustomInstructions ? 'Hide' : 'Add'} custom instructions
          </button>

          {showCustomInstructions && (
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Custom Instructions (Optional)
              </label>
              <textarea
                value={customInstructions}
                onChange={(e) => setCustomInstructions(e.target.value)}
                placeholder="E.g., Mention our new happy hour promotion..."
                className="w-full px-3 py-2 border border-gray-300 rounded-md"
                rows={3}
              />
            </div>
          )}

          <button
            onClick={handleGenerate}
            disabled={isGenerating}
            className={`w-full py-3 px-4 rounded-md font-medium transition-colors ${
              isGenerating
                ? 'bg-gray-400 cursor-not-allowed'
                : 'bg-blue-600 hover:bg-blue-700 text-white'
            }`}
          >
            {isGenerating ? 'Generating...' : 'Generate AI Response'}
          </button>

          {isGenerating && generationState.isGenerating && (
            <div className="mt-4 space-y-2">
              <div className="flex justify-between text-sm">
                <span className="text-gray-600">{generationState.currentStep}</span>
                <span className="text-gray-600">{generationState.progress}%</span>
              </div>
              <div className="w-full bg-gray-200 rounded-full h-2">
                <div
                  className="bg-blue-600 h-2 rounded-full transition-all duration-500"
                  style={{ width: `${generationState.progress}%` }}
                />
              </div>
            </div>
          )}
        </div>
      ) : (
        <div className="text-center py-4">
          <p className="text-gray-600 mb-4">
            An AI response has already been generated.
          </p>
          <button
            onClick={() => setShowCustomInstructions(true)}
            className="text-blue-600 hover:text-blue-700"
          >
            Generate a new version
          </button>
        </div>
      )}
    </div>
  );
};
```

#### 4.2 AIResponseViewer Component

Create `src/components/ai/AIResponseViewer.tsx`:

```typescript
import React, { useState } from 'react';
import { AIResponse } from '../../types';
import { formatDistanceToNow } from '../../utils/dateUtils';

interface AIResponseViewerProps {
  response: AIResponse;
  onEdit: (text: string) => void;
  onApprove: () => void;
  onSend: () => void;
}

export const AIResponseViewer: React.FC<AIResponseViewerProps> = ({
  response,
  onEdit,
  onApprove,
  onSend
}) => {
  const [isEditing, setIsEditing] = useState(false);
  const [editedText, setEditedText] = useState(response.responseText);

  const handleSave = () => {
    onEdit(editedText);
    setIsEditing(false);
  };

  const sentimentColor = {
    positive: 'text-green-600',
    neutral: 'text-yellow-600',
    negative: 'text-red-600'
  };

  const sentimentBg = {
    positive: 'bg-green-50',
    neutral: 'bg-yellow-50',
    negative: 'bg-red-50'
  };

  return (
    <div className="bg-white rounded-lg shadow">
      {/* Header */}
      <div className="px-6 py-4 border-b">
        <div className="flex justify-between items-start">
          <div>
            <h3 className="text-lg font-semibold">AI Generated Response</h3>
            <p className="text-sm text-gray-500 mt-1">
              Version: {response.version.split('_')[0]} • 
              Generated {formatDistanceToNow(response.createdAt)} ago
            </p>
          </div>
          <div className="flex items-center space-x-2">
            <span className={`px-3 py-1 rounded-full text-sm ${
              response.status === 'draft' ? 'bg-gray-100' :
              response.status === 'approved' ? 'bg-green-100 text-green-700' :
              'bg-blue-100 text-blue-700'
            }`}>
              {response.status.charAt(0).toUpperCase() + response.status.slice(1)}
            </span>
          </div>
        </div>
      </div>

      {/* Sentiment Analysis */}
      <div className={`px-6 py-3 ${sentimentBg[response.sentimentAnalysis.sentiment]}`}>
        <div className="flex items-center justify-between">
          <div>
            <span className="text-sm font-medium">Sentiment Analysis:</span>
            <span className={`ml-2 font-semibold ${sentimentColor[response.sentimentAnalysis.sentiment]}`}>
              {response.sentimentAnalysis.sentiment.toUpperCase()}
            </span>
            <span className="ml-4 text-sm text-gray-600">
              Satisfaction: {(response.sentimentAnalysis.satisfactionScore * 100).toFixed(0)}%
            </span>
          </div>
          {response.sentimentAnalysis.keyIssues.length > 0 && (
            <div className="text-sm">
              <span className="text-gray-600">Key issues: </span>
              <span className="font-medium">
                {response.sentimentAnalysis.keyIssues.join(', ')}
              </span>
            </div>
          )}
        </div>
      </div>

      {/* Response Text */}
      <div className="p-6">
        {isEditing ? (
          <div className="space-y-4">
            <textarea
              value={editedText}
              onChange={(e) => setEditedText(e.target.value)}
              className="w-full px-3 py-2 border border-gray-300 rounded-md"
              rows={8}
            />
            <div className="flex justify-end space-x-2">
              <button
                onClick={() => {
                  setEditedText(response.responseText);
                  setIsEditing(false);
                }}
                className="px-4 py-2 border border-gray-300 rounded-md hover:bg-gray-50"
              >
                Cancel
              </button>
              <button
                onClick={handleSave}
                className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700"
              >
                Save Changes
              </button>
            </div>
          </div>
        ) : (
          <div>
            <div className="whitespace-pre-wrap text-gray-800">
              {response.responseText}
            </div>
            {response.status === 'draft' && (
              <button
                onClick={() => setIsEditing(true)}
                className="mt-4 text-blue-600 hover:text-blue-700 text-sm"
              >
                Edit Response
              </button>
            )}
          </div>
        )}
      </div>

      {/* Actions */}
      {response.status === 'draft' && !isEditing && (
        <div className="px-6 py-4 bg-gray-50 border-t flex justify-end space-x-3">
          <button
            onClick={onApprove}
            className="px-4 py-2 bg-green-600 text-white rounded-md hover:bg-green-700"
          >
            Approve Response
          </button>
        </div>
      )}

      {response.status === 'approved' && (
        <div className="px-6 py-4 bg-gray-50 border-t flex justify-between items-center">
          <p className="text-sm text-gray-600">
            Approved by {response.approvedBy} • {formatDistanceToNow(response.approvedAt!)} ago
          </p>
          <button
            onClick={onSend}
            className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700"
          >
            Send Response
          </button>
        </div>
      )}

      {/* Generation Metadata */}
      <div className="px-6 py-3 bg-gray-50 border-t text-xs text-gray-500">
        <div className="flex items-center justify-between">
          <span>
            Model: {response.generationMetadata.modelId} • 
            Prompt v{response.generationMetadata.promptVersion} • 
            Temperature: {response.generationMetadata.temperature}
          </span>
          <span>
            Generated in {response.generationMetadata.generationTimeMs}ms
          </span>
        </div>
        {response.generationMetadata.customInstructions && (
          <div className="mt-2">
            Custom instructions: {response.generationMetadata.customInstructions}
          </div>
        )}
      </div>
    </div>
  );
};
```

### Task 5: Update Feedback Detail Page (1 hour)

Update the existing feedback detail page to include AI response functionality:

```typescript
// In FeedbackDetail.tsx, add:

import { AIResponseGenerator } from '../../components/ai/AIResponseGenerator';
import { AIResponseViewer } from '../../components/ai/AIResponseViewer';
import { ResponseHistory } from '../../components/ai/ResponseHistory';
import { useAIResponse } from '../../hooks/useAIResponse';

// Inside component:
const {
  response,
  history,
  generateResponse,
  regenerateResponse,
  isGenerating,
  hasResponse
} = useAIResponse(feedbackId);

// In the render section, add a new tab or section:
<div className="mt-8">
  <h2 className="text-xl font-semibold mb-4">AI Response</h2>
  
  {!hasResponse ? (
    <AIResponseGenerator
      feedbackId={feedbackId}
      onGenerate={generateResponse}
      isGenerating={isGenerating}
      hasExistingResponse={false}
    />
  ) : (
    <div className="space-y-4">
      <AIResponseViewer
        response={response!}
        onEdit={(text) => console.log('Edit:', text)}
        onApprove={() => console.log('Approve')}
        onSend={() => console.log('Send')}
      />
      
      {history && history.length > 0 && (
        <ResponseHistory
          history={history}
          onSelectVersion={(version) => console.log('Select:', version)}
        />
      )}
      
      <RegenerationModal
        onRegenerate={regenerateResponse}
        isRegenerating={isGenerating}
        previousVersion={response!.version}
      />
    </div>
  )}
</div>
```

### Task 6: Create Tests (1-2 hours)

Create `src/tests/ai-response.test.tsx`:

```typescript
describe('AI Response Generation', () => {
  test('generates response successfully', async () => {
    // Test generation flow
  });

  test('displays loading state during generation', async () => {
    // Test loading animations
  });

  test('handles generation errors gracefully', async () => {
    // Test error handling
  });

  test('allows editing draft responses', async () => {
    // Test edit functionality
  });

  test('prevents editing approved responses', async () => {
    // Test status-based permissions
  });

  test('displays response history', async () => {
    // Test history display
  });

  test('regenerates with custom instructions', async () => {
    // Test regeneration
  });

  test('shows sentiment analysis correctly', async () => {
    // Test sentiment display
  });

  test('updates status on approval', async () => {
    // Test approval flow
  });

  test('disables send until approved', async () => {
    // Test send restrictions
  });

  // Add 5+ more tests for comprehensive coverage
});
```

## 🧪 TESTING REQUIREMENTS

### Component Tests (Minimum 15)
1. ✅ AI response generator renders
2. ✅ Generation triggers API call
3. ✅ Loading states display correctly
4. ✅ Progress animation works
5. ✅ Custom instructions toggle
6. ✅ Response viewer displays data
7. ✅ Edit mode toggles correctly
8. ✅ Sentiment analysis displays
9. ✅ Status badge shows correctly
10. ✅ Approve button works
11. ✅ Send button enables after approval
12. ✅ History displays versions
13. ✅ Regeneration modal works
14. ✅ Error messages display
15. ✅ Mock API returns data

### Integration Tests
1. Full generation flow (request → response → display)
2. Edit and save flow
3. Regeneration with history tracking
4. Error recovery scenarios

### Manual Testing Checklist
- [ ] Generate response for negative feedback
- [ ] Generate response for positive feedback
- [ ] Add custom instructions
- [ ] Edit generated response
- [ ] View response history
- [ ] Regenerate response
- [ ] Approve response
- [ ] Test error scenarios
- [ ] Test loading animations
- [ ] Test on mobile viewport

## ⚡ PERFORMANCE REQUIREMENTS

- Initial render: < 200ms
- API response handling: < 100ms overhead
- Smooth animations (60 FPS)
- Responsive on mobile devices

## 🛡️ ERROR HANDLING

1. **Network Errors**: Show retry button with helpful message
2. **Generation Timeout**: Show timeout message after 30s
3. **Invalid Response**: Fallback to error state
4. **Rate Limiting**: Show rate limit message
5. **Permission Errors**: Show appropriate access denied message

## 📋 DELIVERABLES

1. ✅ AI API service implemented
2. ✅ React hooks for AI responses
3. ✅ 4+ AI components created
4. ✅ Feedback detail page updated
5. ✅ 15+ component tests passing
6. ✅ Loading states polished
7. ✅ Error handling comprehensive
8. ✅ Mobile responsive

## 🚀 SUCCESS CRITERIA

- [ ] Can generate AI response from UI
- [ ] Loading states are smooth and informative
- [ ] Can edit and save draft responses
- [ ] Can view response history
- [ ] Can regenerate with custom instructions
- [ ] All tests pass (minimum 15)
- [ ] Works on mobile devices
- [ ] Error scenarios handled gracefully

## 📝 NOTES

- Use the mockup as reference for UI design
- Maintain consistency with existing components
- Show realistic loading times (2-3s for generation)
- Keep the mock API functional for development
- Focus on user experience over complex features

## 🔗 REFERENCES

- Mockup reference: `/encore_archive/mockup-backup/src/components/`
- Existing hooks: `src/hooks/`
- API client: `src/services/apiClient.ts`
- Type definitions: `src/types/index.ts`

---

**Estimated Time**: 2-3 days
**Dependencies**: Phase 5A backend complete
**Next Phase**: Phase 6 (Email Integration)
