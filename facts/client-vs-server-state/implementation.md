## Execution: The Hybrid Pattern

In real-world applications, Client and Server state must interact without merging. Below is a unified Blog Post Editor pattern using both libraries correctly.

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { create } from 'zustand';

// 1. CLIENT STATE: Manages the temporary draft UI
const useEditorStore = create((set) => ({
  draftTitle: '',
  draftContent: '',
  isDirty: false,
  setDraft: (title, content) => set({ draftTitle: title, draftContent: content, isDirty: true }),
  reset: () => set({ draftTitle: '', draftContent: '', isDirty: false })
}));

export function BlogPostEditor({ postId }) {
  const queryClient = useQueryClient();
  const { draftTitle, draftContent, isDirty, setDraft, reset } = useEditorStore();

  // 2. SERVER STATE: Fetches the source of truth
  const { data: post, isLoading } = useQuery({
    queryKey: ['posts', postId],
    queryFn: () => fetch(`/api/posts/${postId}`).then(r => r.json())
  });

  // 3. MUTATION: Syncs client state back to the server
  const { mutate: savePost } = useMutation({
    mutationFn: (payload) => fetch(`/api/posts/${postId}`, {
      method: 'PUT',
      body: JSON.stringify(payload)
    }),
    onSuccess: () => {
      // Invalidate the cache to trigger a background refetch
      queryClient.invalidateQueries({ queryKey: ['posts', postId] });
      // Reset the local UI state
      reset();
    }
  });

  if (isLoading) return <Spinner />;

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      savePost({ title: draftTitle || post.title, content: draftContent || post.content });
    }}>
      <input
        value={draftTitle || post.title}
        onChange={(e) => setDraft(e.target.value, draftContent)}
      />
      <button disabled={!isDirty}>Save to Server</button>
    </form>
  );
}
```
