# One shot

This is one real shot from `kate-canvas.json`: **Video Generator 1**. Copy this shape a few times. Do not copy the rest of the board.

## Graph

- **Video node:** `videoGenerator`, model `seedance-2.0-i2v` on that board. New runs use `seedance-2.5` (image-to-video). 9 seconds, 9:16, 720p, audio off.
- **Start still:** image upload **Image Upload 17**, wired `image-out` → `image-in-0`. That frame is the lock for the people, the purple product bag, the rice field, and the framing.
- **End still:** image upload **Image Upload 14**, wired `image-out` → `lastframe-in`.
- **Prompt:** stored on the video node in the export. When you build a new shot, put this directing copy in the shot's **one text node** and wire `text-out` → `text-in`.

No image-generator keyframe on this shot. Add one only when the user does not already have a frame that combines character, product, and place.

## What the prompt locks

It tells Seedance to treat the uploaded image as the exact visual reference for the **characters**, the **purple bag** (the product), the **environment**, the **composition**, and the overall look. It then asks for **one continuous shot with no cuts**: a shirtless man in a straw hat snatches the bag from a helmeted woman in a scorching Thai rice field, digs through it without showing the contents, and the camera rotates into an over-the-shoulder medium shot as he throws the bag and shouts one Thai line.

The timeline stays inside the 9 seconds (snatch, search, rotate, shout, hold on the face). Style is an exaggerated Thai TV comedy commercial. The close repeats the lock: keep the people's appearance, the bag color and design, the rice-field setting, and the framing. No subtitles, no music, no scene cuts.

## Build the next shot the same way

1. Product upload (the bag, pack, or logo).
2. Reference stills for character, place, and framing. Wire the start frame first so it is `@Image1`.
3. Optional keyframe image generator if those stills still need to be composed.
4. One text node whose prompt locks character, product, place, and framing, then describes one continuous action with a timeline.
5. One Seedance video node. "Make it slower" edits this node (prompt and duration) and re-runs it.
