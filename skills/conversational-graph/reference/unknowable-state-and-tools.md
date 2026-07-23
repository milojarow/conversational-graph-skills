# State the bot cannot know — forbid it, then build the read-only tool

Every bot has questions its user will obviously ask and it has **no way to answer**: is that slot free, is the item in stock, where is my order, has it shipped. This file covers the two stages of handling them: the prohibition that stops the invention, and the read-only tool that actually closes the gap.

## Without a tool to check it, the model INVENTS the fact — in a confident tone

**Wall:** asked *"can I come Thursday at 4pm?"* the bot answered *"Sure, Thursday at 4:00 pm we do have space."* It had **no scheduling tool at all** — it only files requests that the business confirms afterwards. Nobody asked it to lie: the question had yes/no shape, there was no way to look it up, and the model filled the hole with the cooperative answer. In an appointments business that sends a customer to the door expecting a slot that may not exist.

This is a cousin of *"confirm a side-effect only on the tool's REAL result"* ([state-and-small-models.md](state-and-small-models.md)) but **harder to catch**: there, a tool exists and can fail; here there is **no tool at all**, so there is no result to contradict the model and no error to log. It passes silently until somebody reads a transcript.

**Rule: for every state question the bot cannot look up, put an explicit prohibition in BASE *and* give it the true alternative it CAN say.** Without the alternative the model chooses between sounding useless and inventing — and it invents.

> AVAILABILITY: you have no access to the calendar. NEVER claim there is space/room at a specific day or hour, nor that "that works" at that time. What you CAN say: whether the hour falls inside business hours, and that the business confirms availability when it contacts them. What you record is a REQUEST, never a confirmed reservation.

With that rule in place, three escalating questions (including an imperative *"just hold it for me"*) all distinguish *falls inside opening hours* from *is free*, and none promises the slot.

**How to hunt the rest of these gaps before a customer does:** list the tools the bot HAS, then ask which obvious user questions fall outside them — stock, availability, order status, delivery time, "has it arrived yet". Each gap is one BASE rule plus its alternative. Test each with yes/no questions **and** with an imperative ("hold it", "reserve it") — the imperative is what pushes the model hardest into asserting.
