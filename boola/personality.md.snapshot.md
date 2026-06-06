# Boola — Personality / Speech Bubble Library

This file is the content source for Boola's speech bubbles (T55, Update 7). Boola loads it at startup, parses by `## category_name` headers, and randomly picks one line per fire.

## Tone rules

- Sales-rep coworker who's seen everything. Confident, witty, lightly sassy.
- Never corporate ("Great work, team!"). Never therapy-speak ("How are we feeling?"). Never AI-slop ("I'm here to help you crush it!").
- Three to ten words per line. Stop. Bubbles shouldn't be paragraphs.
- Use ONE emoji per line max. Skip if it doesn't add anything.
- Punctuation: occasional period, occasional ellipsis, no exclamation marks except for legit celebration moments.
- Hold standards. If a line wouldn't make a real sales rep crack a smile, cut it.

## Maintenance

- Boola picks a random line per category at fire time.
- Tracks last-fired line per category in memory so you never see the same one twice in a row.
- Add new lines anytime — Boola re-reads on relaunch.
- One blank line between categories. Lines themselves are unprefixed (no bullets).

---

## sent_email

Out the door.
Inbox roulette begins. 🎰
Cold email, warm intentions.
Pixels traveling at the speed of business.
Hope they read between the lines.
That subject line did some heavy lifting.
You wrote it. I sent it. Now we wait.
Ball's in their court.
Filled the truck. 🚚
Email shipped. Wallet incoming.
One step closer to "yes."
That's outreach.

## cold_first_contact

First impression: deployed.
Cold to warm in 3, 2, 1...
You don't ask, you don't get.
Reaching out beats reaching in.
Hi-stranger-buy-stuff energy.
Stranger danger, in reverse.
Their inbox just met you.

## task_complete

One down. Who's next?
Boom. ✅
Strike one off the list.
Less list, more lift.
That felt good.
Productivity has entered the chat.
Quick win counts as a win.
Closer to clocking out.
The list is shrinking. Beautiful.
One less thing to dream about tonight.

## leads_loaded

Today's lineup is ready.
Ten fresh ones, served warm.
Pipeline's not empty.
Names. Numbers. Money.
Your morning, deserved.
Coffee + leads. Now go.
Fresh batch. Don't let them go cold.
The list works. Now you do.

## deal_won

🍾 That's how it's done.
Closed. Booked. Banked. 💰
Add it to the trophy shelf.
Champagne expense report?
Quota? What quota.
The dance. Do it. 💃
This is why we cold called Tuesday.
That's a paycheck.

## call_list_add

Added. They're on the docket.
One for the pile.
Tomorrow's wins start here.
Loaded the chamber.
That's a name with potential.

## call_start

Headphones on. Game face on.
Showtime. 🎙️
Voice ready, intent clear.
The mic is hot.
Let's hear them out.

## call_end

Wrapped. Now: notes.
Cool. Cool. Notes time.
Don't skip the follow-up.
What did they actually say?
Quick — log it before you forget.

## streak_5_todos

Five down. Who are you. 😎
On a tear today.
This is what momentum looks like.
Productivity gremlin mode: ON.
The to-do list is afraid of you.

## morning_greeting

Morning. Leads are warm.
Coffee. Leads. Money.
Quota doesn't close itself.
Today's a closing day.
Pipeline's hungry. Feed it.
Top of the morning, rep.
Let's make it weird (for the competitors).
Ready to ruin some inboxes?

## welcome_back

Welcome back. Miss me?
Pipeline didn't move while you were out. Get to it.
Inbox waits for no one.
Where we left off: half a quota.
Took a nap?

## no_response_breakup

Sometimes silence speaks. Sometimes it's just lunch.
Closure email loading...
The last touch IS a touch.
"Just checking in" — but cooler.

## proposal_sent

Numbers are out. Now: pressure.
Proposal lives. Let's see if it kicks.
This is the part where they pretend to think about it.
You priced it right. Trust the work.

## after_call

Notes first, then breathe.
Capture it while it's fresh.
The follow-up email is half the call.

## new_lead_high_confidence

This one's a layup. 🏀
Confidence score = 90+. Pounce.
Don't overthink this one. Just call.
Right vertical, right time, right now.

## new_lead_low_confidence

Long shot, but they all start somewhere.
Worth a touch. Quick one.
Even cold leads warm up.

## ai_overloaded

The robots are tired. One sec.
Anthropic's having a moment.
Queue's deep. Stay patient.
Brain buffering...

## end_of_day

Pencils down. Tomorrow exists.
You did your reps today.
Pipeline persists. So do you.
Log off. Touch grass. 🌱
Tomorrow's leads are already cooking.

## generic_idle

(intentionally blank — idle bubbles are banned by the Mascot Rule)
