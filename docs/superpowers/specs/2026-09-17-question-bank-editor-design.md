# Question Bank Editor — Design Spec

Date: 2026-09-17

## Problem

The game (`index.html`) has its 10 quiz questions hardcoded in a `QUESTIONS` JS array. The user (a teacher) wants to add, edit, delete, and reorder questions through the UI instead of editing code.

## Data Model

Each question:

```js
{
  id: string,          // stable id, e.g. "q_" + timestamp + random suffix
  stage: "KHỞI ĐỘNG" | "TĂNG TỐC" | "VỀ ĐÍCH",
  time: number,        // seconds to answer
  q: string,           // question text
  a: [string, string, string, string], // 4 answers
  correct: 0|1|2|3,    // index of correct answer
  ex: string           // explanation shown after reveal
}
```

`id` is added (didn't exist before) so list items, edit targets, and reorder operations have a stable key independent of array position.

## Persistence

- `localStorage` key `millionaire_questions_v1` holds the full JSON array (source of truth used by the running game).
- On first load (key absent or unparsable), seed from the existing 10 hardcoded questions (kept in code as `DEFAULT_QUESTIONS`), then write them to `localStorage`.
- Every add/edit/delete/reorder immediately persists the full array back to `localStorage`.
- **Export**: button builds a `Blob` of the current array as pretty JSON and triggers a download (`cau-hoi-trieu-phu.json`).
- **Import**: file input reads a JSON file, validates it's an array of well-formed question objects (each has non-empty `q`, exactly 4 non-empty `a`, `correct` in 0-3, `stage`, numeric `time > 0`); on success, confirm with the user ("Nhập N câu hỏi sẽ THAY THẾ toàn bộ ngân hàng hiện tại. Tiếp tục?"), then replace and persist. Invalid file → alert with a short error, no change.
- **Reset to default**: confirm ("Khôi phục 10 câu hỏi mặc định sẽ THAY THẾ toàn bộ ngân hàng hiện tại. Tiếp tục?"), then replace with a fresh deep copy of `DEFAULT_QUESTIONS` (each getting a new `id`), persist.

## New Screen: `#manage-screen`

Reachable via a new "QUẢN LÝ CÂU HỎI" button on the welcome screen; has its own "← Quay lại" button back to welcome. Only reachable from welcome (not mid-game).

Layout:
- Header: title + question count + back button.
- Toolbar: "+ Thêm câu hỏi", "Xuất JSON", "Nhập JSON" (hidden file input triggered by a button), "Khôi phục mặc định".
- List: one card per question, in play order, showing:
  - Stage badge + time (e.g. "TĂNG TỐC · 15s")
  - Question text (truncated if long)
  - Correct answer highlighted inline
  - Buttons: ↑ (move up), ↓ (move down), Sửa (edit), Xóa (delete, with confirm)
- Add/Edit form (shown inline, replacing the toolbar+list area, or appended above the list — implementation detail, not user-visible distinction that matters): stage `<select>` (3 fixed options), time `<input type=number>` (default 15), question `<textarea>`, 4 answer `<input type=text>`, radio group to mark which of the 4 is correct, explanation `<textarea>`, Save/Cancel buttons.
  - Validation before save: question non-empty, all 4 answers non-empty, one answer marked correct, time is a positive integer. Show inline error text and block save if invalid.
- Empty state: if list is empty, show a message ("Chưa có câu hỏi nào. Bấm “+ Thêm câu hỏi” để bắt đầu.") instead of an empty list.

Reordering is array-index swap (move up = swap with previous element), persisted immediately.

## Welcome Screen Changes

- Add "QUẢN LÝ CÂU HỎI" button (secondary style, next to/below "BẮT ĐẦU").
- `.welcome-note` text becomes dynamic: "Trò chơi gồm N câu hỏi, chia làm 3 chặng độ khó tăng dần" where N = current question count. If N === 0, note instead reads "Chưa có câu hỏi nào — hãy vào “Quản lý câu hỏi” để thêm trước khi bắt đầu."
- "BẮT ĐẦU" button is `disabled` when the question bank is empty (0 questions), re-enabled reactively when the user adds a question and returns to welcome.

## In-Game Changes

- `QUESTIONS` becomes a mutable variable loaded from storage at startup (and reloaded when returning to welcome from manage screen, in case it changed) instead of a hardcoded const array.
- `q-num` label: `"CÂU " + (i+1) + "/" + QUESTIONS.length` (was hardcoded `"/10"`).
- Game plays through all current questions in stored order (no random subset selection — explicitly decided against for this iteration).
- All other game logic (timer, 50:50, add time, team scoring, remote sync) is unaffected; `QUESTIONS.length` already drives the "last question" check (`finishGame`), so no change needed there beyond removing the hardcoded label.

## Out of Scope

- Remote control (`remote.html` / `server.js` / Firebase channel) is unaffected — question management is local-only, not exposed to the phone remote.
- No random-subset-per-game selection (explicitly rejected).
- No drag-and-drop reordering (up/down buttons are sufficient).
- No per-question stage renaming beyond the 3 existing fixed stage names.

## Risks / Notes

- `localStorage` is per-browser/per-device; export/import JSON is the mechanism for moving a question bank between devices or backing it up, per user's explicit choice.
- Existing default questions remain in code as the reset/seed source, so "Khôi phục mặc định" always works even after heavy editing.
