# Delegated curve review

Use this only when the user has explicitly delegated curve approval (SKILL.md, step 7). Give each
product line to a fresh reviewer agent that did not prepare the import. The reviewer judges the
source curves only; it calls no EKX tool and edits nothing.

## What to give the reviewer

- Review sheets: every rating of the line drawn on log-log axes with equal-length decades, at most
  about 12 ratings per sheet. Draw minimum melt solid, total clear dashed and an average curve
  dotted, using the stored points joined by straight segments, as EKX evaluates them.
- Per-rating numbers: the first point as a multiple of rating; the time at 2x, 5x, 10x and 20x
  rating for each curve (none when the curve does not reach that current); and, for a melt/clear
  pair, how many total-clear points are faster than minimum melt at the same current.
- The planned records: rating, voltage, speed, source revision, and whether each is a pair or an
  average curve.

## Accept a rating when all of these hold

1. Each curve falls as current rises. A vertical step (two points at one current) or a small dip of
   a few percent near the slow top of a fuse curve is acceptable; EKX reads it conservatively.
   Reject a reversal or loop large enough to change the curve's shape.
2. For a melt/clear pair, total clear is never faster than minimum melt at the same current, and
   the two curves have the same general shape.
3. The rating sits in order among its neighbours: at the same time, a larger rating needs more
   current. Curves of the same type for adjacent ratings do not cross, except in the last few
   points near the fastest published time.
4. The rating's shape and multiples match the line's pattern. Values about ten times off their
   neighbours suggest a wrong label or unit.

Do not judge a rating against a fixed time at a multiple of rating: time-delay rules differ by fuse
class (Class J and RK time-delay fuses hold at least 10 s at 500%; Class CC, Class G and
supplementary fuses are defined at 200%), and a library's speed label is not a fuse class. Do not
reject a rating only because its curve covers part of the range.

## Output and use

The reviewer returns accept or reject for every planned rating, with a one-sentence reason for each
rejection. Leave rejected ratings out of the build, accept the curves of accepted ratings with their
current review fingerprints, and report the reviewer's decisions and reasons to the user.
