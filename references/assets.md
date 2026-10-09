# Specifying a wide illustration

When a full-width band (a header, a hero) is filled by a phone-ratio illustration, wide screens either
upscale it until only a strip shows, or leave bars beside it. Ask the designer for a wide variant.

## Brief

- **Format**: vector PDF with *Preserve Vector Data* (one file, any size), or PNG @2x / @3x.
- **Size**: band height × the widest width to cover, plus margin. For a 250 pt band up to an iPad Pro
  13" in landscape (1376 pt): **1400 × 250 pt** (5.6:1).
- **Widths it must work at** (same height): Duo unfolded ~780 pt (~3:1), iPad portrait 744–1032 pt
  (3–4:1), iPad landscape up to 1376 pt (5.5:1). Phones and the folded Duo keep the phone artwork.
- **Safe zone**: the image fills the band's height and is anchored to one side, so a narrow screen sees
  only that side. Put the key elements in the ~700 pt nearest the anchor; the far side extends simply
  (sky, ground) and sits behind text.
- **Hidden margins**: whatever is tucked under a card below, and ~25 pt under the status bar at the top.
- **Overlays**: note any element drawn on top (a mascot in a corner) so nothing important sits under it.

## In code

Switch by the band's measured aspect ratio, not by size class or device — see `patterns.md` §3. A
threshold around 1.8:1 suits a 1.6:1 phone image.
