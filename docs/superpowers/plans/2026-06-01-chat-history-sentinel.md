# Chat History Sentinel — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix chat close back navigation — separate UI close from browser traversal using a module-level sentinel counter.

**Architecture:** Module-level `chatHistorySentinel` tracks pushState/close pairs. Every mount/room-change does `pushState + sentinel++`. Every controlled close does `replaceState + sentinel--`. Hardware back navigates within sentinel range without closing chat.

**Tech Stack:** React (ChatWindow.tsx), vanilla History API (pushState/replaceState/popstate).

---

## File Map

- `frontend/src/components/ChatWindow.tsx` — single file, 3 targeted changes (lines ~21, ~234, ~240, ~264)

---

## Task 1: Add Sentinel Counter

**File:** `frontend/src/components/ChatWindow.tsx`

- [ ] **Step 1: Add module-level sentinel counter after imports**

After line 21 (blank line before `interface ChatWindowProps`), add:

```javascript
// History sentinel: counts active #chat pushState entries.
// Increment on mount/room-change (pushState). Decrement on controlled close (replaceState).
// Cleanup does NOT touch this — unmount via popstate is handled separately.
let chatHistorySentinel = 0;
```

Run: `git diff frontend/src/components/ChatWindow.tsx | head -20`
Expected: `+let chatHistorySentinel = 0;` added after imports

- [ ] **Step 2: Commit**

```bash
git add frontend/src/components/ChatWindow.tsx
git commit -m "feat(chat): add chatHistorySentinel module-level counter

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2: Add closeChatWithHistory Helper

**File:** `frontend/src/components/ChatWindow.tsx`

- [ ] **Step 1: Add closeChatWithHistory helper before the history effect (before line 240)**

After the existing `onBackRef` effect (lines 234-238) and before the history effect (line 240), add:

```javascript
// Controlled chat close: removes #chat via replaceState, decrements sentinel,
// then calls parent's onBack to unmount this component.
// Called by both UI back button and popstate when sentinel === 1.
const closeChatWithHistory = () => {
  if (chatHistorySentinel <= 0) {
    // Safety guard: if sentinel is already 0, just close without history manipulation
    onBackRef.current();
    return;
  }
  // Remove #chat from URL without triggering popstate
  window.history.replaceState(null, '', window.location.pathname + window.location.search);
  chatHistorySentinel--;
  onBackRef.current();
};
```

Run: `grep -n "closeChatWithHistory" frontend/src/components/ChatWindow.tsx`
Expected: `237:const closeChatWithHistory = () => {`

- [ ] **Step 2: Commit**

```bash
git add frontend/src/components/ChatWindow.tsx
git commit -m "feat(chat): add closeChatWithHistory helper

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3: Replace History Effect and handleHardwareSafeBack

**File:** `frontend/src/components/ChatWindow.tsx`

- [ ] **Step 1: Replace the history useEffect (lines 240-262)**

Replace the entire "// Android Geri Tuşu / Yandan Kaydırma Kontrolü" effect with:

```javascript
  // Android Geri Tuşu / Yandan Kaydırma Kontrolü
  useEffect(() => {
    if (!selectedRoom?.id) return;

    // Her oda değişikliğinde pushState yap — sentinel'i her zaman artır
    window.history.pushState(null, '', window.location.pathname + window.location.search + '#chat');
    chatHistorySentinel++;

    const handlePopState = () => {
      // Donanım geri tuşu veya swipe back — browser zaten history'e back yaptı
      // Sentinel > 1 → başka #chat entry var, sadece azalt, chat açık kalsın
      // Sentinel === 1 → son #chat entry tüketildi, chat'i kapat
      if (chatHistorySentinel > 1) {
        chatHistorySentinel--;
      } else if (chatHistorySentinel === 1) {
        chatHistorySentinel--;
        closeChatWithHistory();
      }
      // sentinel < 1: zaten 0 veya negatif, bir şey yapma
    };

    window.addEventListener('popstate', handlePopState);

    return () => {
      window.removeEventListener('popstate', handlePopState);
      // NOT: cleanup'ta sentinel AZALTMIYORUZ.
      // Unmount popstate üzerinden zaten handle ediliyor (sentinel === 1 durumu).
      // currentRoom null olmadan ChatWindow unmount olmaz — tüm unmount'lar
      // closeChatWithHistory üzerinden sentinel-- ile birlikte gerçekleşir.
    };
  }, [selectedRoom?.id]);
```

Run: `grep -n "chatHistorySentinel" frontend/src/components/ChatWindow.tsx`
Expected: 4 occurrences — definition, closeChatWithHistory, pushState effect, popstate handler

- [ ] **Step 2: Replace handleHardwareSafeBack (lines 264-272)**

Replace:
```javascript
  // UI içerisindeki geri tiklama fonksiyonu.
  // Race condition oluşmaması için doğrudan React stateni değiştirmek yerine tarayıcının History API'sini kullanıyoruz.
  const handleHardwareSafeBack = () => {
    if (window.location.hash === '#chat') {
      window.history.back(); // Bu işlem popstate'i tetikler ve unmount handlePopState üzerinden güvenle yapılır.
    } else {
      onBack();
    }
  };
```

With:
```javascript
  // UI içerisindeki geri butonu — her zaman chat'i kapatır (controlled close)
  // replaceState kullanır (popstate tetiklemez), sentinel azaltılır
  const handleHardwareSafeBack = () => {
    closeChatWithHistory();
  };
```

Run: `grep -n "handleHardwareSafeBack" frontend/src/components/ChatWindow.tsx`
Expected: 2 occurrences — definition and `onBack={handleHardwareSafeBack}` in ChatHeader props

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/ChatWindow.tsx
git commit -m "feat(chat): replace history effect with sentinel counter approach

- Unconditional pushState + sentinel++ on mount/room-change
- popstate: sentinel > 1 = decrement only; sentinel === 1 = closeChatWithHistory
- handleHardwareSafeBack: always calls closeChatWithHistory (no history.back())
- Cleanup does not touch sentinel

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Verification Checklist

After all tasks complete, run through these scenarios manually:

- [ ] **Basic open/close**: Open chat → URL has `#chat` → click UI back → chat closes, `#chat` gone
- [ ] **Hardware back**: Open chat → press hardware back → chat closes
- [ ] **Room switch + hardware back**: Room A → switch to Room B → hardware back once → back to Room A (chat open) → hardware back again → chat closes
- [ ] **Rapid switch**: Switch 3 rooms quickly → spam hardware back → correct room navigation
- [ ] **Spam UI back**: Click UI back 5× rapidly → no crash, chat closed once
- [ ] **Console errors**: DevTools open during all above → no React errors
