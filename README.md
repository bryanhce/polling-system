# Intro
This project is part of Jr Dev SG mentorship program, with the main goal of improving my system design skills

# Prompts

## First prompt
The Protagonist: Momo, an avid gamer and community manager for the guild Aethelgard. 

Momo recently played a highly anticipated video game called Realm of Shadows and was deeply frustrated by its new loot-sharing mechanics. He wanted to prove to the developers that the community disliked it too, but existing poll tools were either too rigid, too expensive, or lacked the features he needed. Thus, Momo decided to build his own polling platform: Aethelgard Voice.

Momo just wants a minimal viable product. He needs to create a simple poll asking, "Do you like the new loot system in Realm of Shadows?" and let his guildmates submit their votes. He doesn't care if people vote multiple times yet; he just needs the numbers. However, knowing the internet is full of trolls, he wants to ensure his site can't be easily hijacked.

## Second prompt
- Before sharing the link on the guild's Discord, Momo realizes a broken poll would be embarrassing. He needs to make sure the user journey, from clicking the link to seeing the results, works flawlessly. He also starts thinking about weird scenarios: what if someone submits a vote without selecting an option? (Engineering Practice: Either unit tests like vitest or playwright e2e tests, Business Logic: Understanding and handling edge cases (e.g., empty submissions, expired links, malformed data)).

- Momo wants to host a "watch party" on Twitch where he displays the poll on screen, and viewers can watch the bars move in near real-time as the community votes. Manual refreshing won't cut it anymore. (Live Polling: Near real-time updates of vote counts.)