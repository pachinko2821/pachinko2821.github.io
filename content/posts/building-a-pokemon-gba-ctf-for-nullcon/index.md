---
date: '2026-03-04T11:05:33+05:30'
draft: false
title: 'Building a Custom Pokémon GBA ROM for Winja CTF'
cover:
    image: "images/cover.png" 
    alt: "some image that didn't load"
    caption: "modifying pokémon firered"
    responsiveImages: false
tags: ["ctf", "pokémon", "winja", "nullcon"]
keywords: ["game", "gameboy", "ctf", "pokémon"]
description: "Creating GBA pokémon CTF for Winja CTF NullCon"
showFullContent: true
readingTime: true
showFullContent: true
readingTime: true
showToc: true
tocOpen: true
showReadingTime: true
hideSummary: true
---

For Winja CTF Nullcon Goa 2026, I wanted to build a challenge that felt a little different from the usual web, crypto, and reversing problems. Instead of presenting players with a traditional binary or web service, the idea was simple:

**What if the challenge was an actual video game?**

This resulted in a modified version of **Pokémon FireRed**, where the objective was to retrieve a hidden flag embedded inside the game.

Players were given a patched ROM and the goal was straightforward:


```
Beat your rival in Professor Oak's lab
Obtain a special key item from your mother
Exchange it for the flag at the Cerulean Bike Shop
```

Along the way, players could either play the game normally or use emulator tricks and cheats to skip ahead.

Below is a breakdown of how the challenge was designed and implemented.

# Video Solution

A full walkthrough of the challenge can be seen here:

{{< rawhtml >}} 

<video width=100% controls autoplay>
    <source src="files/pokemon-firered-walkthrough.mkv" type="video/webm">
    Your browser does not support the video tag.  
</video>

{{< /rawhtml >}} 

For those who played this at Winja CTF, please note that this was taken very early in the development process, so still has the old flag.

---

# Why Pokémon FireRed?

Modding games has always been a fun thing that I've wanted to try and give a shot. I had never really looked into it before, and honestly it felt like I was trying to avoid the topic.

Fortunately, I came across the **pret project** recently, which changed this completely.

The repository I used was: https://github.com/pret/pokefirered

This project is a **full decompilation of Pokémon FireRed into C code** that can be rebuilt into a working ROM. This makes it surprisingly easy to:

- modify mission scripts
- add new dialogue  
- add new items  
- alter event triggers

And after all this, you can simply rebuild the game with `make`. For a CTF challenge, this was perfect.

---

# Challenge Design

The core gameplay logic of the challenge was intentionally simple.

```
Beat Rival in Oak's Lab
        ↓
Mom gives FLAG_KEY
        ↓
Travel to Cerulean Bike Shop
        ↓
Exchange FLAG_KEY for FLAG
        ↓
XOR decode FLAG with FLAG_KEY
```

Players ultimately receive an **XOR encoded flag**, and the **FLAG_KEY item** contains the key required to decode it.

This meant the challenge had two components:

1. Game mechanics to get the flag
2. Basic crypto to hide the flag value

I wanted to keep the game feeling as natural as possible, with the challenge integrated into it's progression. I also tried to follow the code structure of the original pret developers as closely as possible.

---

# Reproducing the Challenge

If you'd like to try modifying the game yourself, you can recreate the challenge like this:

- Download the patch file: [winjactf.patch](files/winjactf.patch)

- Clone the FireRed decompilation:

```
git clone https://github.com/pret/pokefirered.git
cd pokefirered
```

- Apply the patch:

```
git apply winjactf.patch
```

- Build the ROM:

```
make
```

You can then run the resulting ROM in any Game Boy Advance emulator. I would personally recommend [mGBA](https://mgba.io/) or [gbajs3](https://gba.nicholas-vancise.dev/).

---

# Step 1: Detecting When the Rival is Defeated

The first rival battle happens inside **Professor Oak's Lab** at the very start of the game.

The game normally sets a flag after this battle to indicate that the event has occurred. It doesn't check if you've beat the rival or not.

For the challenge, I ensured that the flag `FLAG_BEAT_RIVAL_IN_OAKS_LAB` was **only set if the player actually wins the battle**.

In `src/battle_setup.c`:
```c
void SetBeatRivalTutorialBattleFlag(void)
{
    FlagSet(FLAG_BEAT_RIVAL_IN_OAKS_LAB);
}

static void CB2_EndTrainerBattle(void)
{
    if (sTrainerBattleMode == TRAINER_BATTLE_EARLY_RIVAL)
    {
        if (IsPlayerDefeated(gBattleOutcome) == TRUE)
        {
            gSpecialVar_Result = TRUE;
            if (sRivalBattleFlags & RIVAL_BATTLE_HEAL_AFTER)
            {
                HealPlayerParty();
            }
            else
            {
                SetMainCallback2(CB2_WhiteOut);
                return;
            }
            SetMainCallback2(CB2_ReturnToFieldContinueScriptPlayMapMusic);
            SetBattledTrainerFlag();
            QuestLogEvents_HandleEndTrainerBattle();
        }
        else
        {
            gSpecialVar_Result = FALSE;
            // Set flag if player wins the battle
            SetBeatRivalTutorialBattleFlag();
            SetMainCallback2(CB2_ReturnToFieldContinueScriptPlayMapMusic);
            SetBattledTrainerFlag();
            QuestLogEvents_HandleEndTrainerBattle();
        }

    }
    // REST OF THE METHOD
}
```
This prevents players from losing the fight and continuing with the story anyway.

Once this flag is set, other scripts can react accordingly.

---

# Step 2: Giving the Player the FLAG_KEY

The next modification happens in the player's house.

After defeating the rival, interacting with the player's mother triggers a new script.

Instead of the normal dialogue, the mother now gives the player a special item `FLAG_KEY` which acts as the decryption key for the flag.

The scripts for a scene that takes place on a map location, and all the dialogues for it are respectively placed in `data/maps/<LOCATION>/scripts.inc` and in `data/maps/<LOCATION>/text.inc`

In `data/maps/PalletTown_PlayersHouse_1F/text.inc`, I added new dialogues for when Mom gives you the `FLAG_KEY` item: 
```c
PalletTown_PlayersHouse_1F_Text_AllBoysLeaveGotKey::
    .string "MOM: …Right.\n"
    .string "All boys leave home someday.\l"
    .string "It said so on TV.\p"
    .string "Oh, yes. Don't forget to take that\n"
    .string "key to the Cerulean City Bike Shop.$"

PalletTown_PlayersHouse_1F_Text_AllGirlsLeaveGotKey::
    .string "MOM: …Right.\n"
    .string "All girls dream of traveling.\l"
    .string "It said so on TV.\p"
    .string "Oh, yes. Don't forget to take that\n"
    .string "key to the Cerulean City Bike Shop.$"
 
PalletTown_PlayersHouse_1F_Text_YouShouldTakeQuickRest::
    .string "MOM: {PLAYER}!\n"
    .string "You defeated {RIVAL} at\p"
    .string "Prof. Oak's lab?!\n"
    .string "Here, have this...\p"
    .string "You should take a quick rest.$"
 
PalletTown_PlayersHouse_1F_Text_LookingGreatTakeCare::
    .string "MOM: Oh, good! You and your\n"
    .string "POKéMON are looking great.\l"
	.string "Take the key to the Bike Shop\p"
    .string "in Cerulean City!\l"
    .string "Take care now!$"
```

And in `data/maps/PalletTown_PlayersHouse_1F/scripts.inc` I added new routes so that the items get given if conditions are met:
```c
PalletTown_PlayersHouse_1F_EventScript_Mom::
	lock
	faceplayer
    // check if ITEM_FLAG_KEY is already given
	goto_if_set FLAG_GOT_FLAG_KEY, PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYou
	goto_if_set FLAG_BEAT_RIVAL_IN_OAKS_LAB, PalletTown_PlayersHouse_1F_EventScript_MomHeal
	checkplayergender
	call_if_eq VAR_RESULT, MALE, PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYouMale
	call_if_eq VAR_RESULT, FEMALE, PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYouFemale
	closemessage
	applymovement LOCALID_MOM, Common_Movement_FaceOriginalDirection
	waitmovement 0
	release
	end
 
PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYou::
	checkplayergender
	call_if_eq VAR_RESULT, MALE, PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYouMaleGotKey
	call_if_eq VAR_RESULT, FEMALE, PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYouFemaleGotKey
	releaseall
	end

PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYouMaleGotKey::
	msgbox PalletTown_PlayersHouse_1F_Text_AllBoysLeaveGotKey
	return

PalletTown_PlayersHouse_1F_EventScript_MomOakLookingForYouFemaleGotKey::
	msgbox PalletTown_PlayersHouse_1F_Text_AllGirlsLeaveGotKey
	return

PalletTown_PlayersHouse_1F_EventScript_MomHeal::
    msgbox PalletTown_PlayersHouse_1F_Text_YouShouldTakeQuickRest
    giveitem ITEM_FLAG_KEY, 1
    // set once when ITEM_FLAG_KEY is given
    setflag FLAG_GOT_FLAG_KEY
    closemessage
    call EventScript_OutOfCenterPartyHeal
    msgbox PalletTown_PlayersHouse_1F_Text_LookingGreatTakeCare
```

I also had to make sure that the scripts would trigger regardless of gender, and the dialogues would feel right.

Once you are given the `FLAG_KEY` the game sets the `FLAG_GOT_FLAG_KEY` flag in game, which is used to trigger alternate dialogues at locations `PalletTown_PlayersHouse_1F` and `CeruleanCity_BikeShop`.

---

# Step 3: Modifying the Bike Shop

The Cerulean Bike Shop already contains a useful mechanic in the original game. Normally, you receive a Bike Voucher, which can be exchanged for a Bicycle. I reused this mechanic to implement the flag exchange.

In `data/maps/CeruleanCity_BikeShop/scripts.inc`,
```c
CeruleanCity_BikeShop_EventScript_Clerk::
	lock
	faceplayer
    // check if ITEM_FLAG is already given
	goto_if_set FLAG_GOT_FLAG, CeruleanCity_BikeShop_EventScript_AlreadyGotFlag
	goto_if_set FLAG_GOT_FLAG_KEY, CeruleanCity_BikeShop_EventScript_ExchangeFlagKey
	goto_if_set FLAG_GOT_BICYCLE, CeruleanCity_BikeShop_EventScript_AlreadyGotBicycle
	goto_if_set FLAG_GOT_BIKE_VOUCHER, CeruleanCity_BikeShop_EventScript_ExchangeBikeVoucher
	showmoneybox 0, 0
	message CeruleanCity_BikeShop_Text_WelcomeToBikeShop
	waitmessage
	multichoice 11, 0, MULTICHOICE_BIKE_SHOP, FALSE
	switch VAR_RESULT
	case 0, CeruleanCity_BikeShop_EventScript_TryPurchaseBicycle
	case 1, CeruleanCity_BikeShop_EventScript_ClerkGoodbye
	case 127, CeruleanCity_BikeShop_EventScript_ClerkGoodbye
	end

CeruleanCity_BikeShop_EventScript_ExchangeFlagKey::
	msgbox CeruleanCity_BikeShop_Text_OhFlagKeyHereYouGo
	msgreceiveditem CeruleanCity_BikeShop_Text_ExchangedKeyForFlag, ITEM_FLAG, 1, MUS_OBTAIN_KEY_ITEM
	additem ITEM_FLAG
	msgbox CeruleanCity_BikeShop_Text_ThankYouComeAgain
    // set once when ITEM_FLAG is given
	setflag FLAG_GOT_FLAG
	release
	end

CeruleanCity_BikeShop_EventScript_AlreadyGotFlag::
	msgbox CeruleanCity_BikeShop_Text_YouCanDecodeTheFlag
	release
	end
```

The shop clerk now checks if the player has `FLAG_KEY` and gives them `FLAG` if thats the case, and then sets `FLAG_GOT_FLAG`. Now, if you talk to the clerk again, he triggers an alternate dialogue to give you a hint on how to decode the flag.

In `data/maps/CeruleanCity_BikeShop/text.inc`,
```c
CeruleanCity_BikeShop_Text_OhFlagKeyHereYouGo::
    .string "Oh, that's…\p"
    .string "A FLAG KEY!\p"
    .string "Okay!\n"
    .string "Here you go!$"

CeruleanCity_BikeShop_Text_YouCanDecodeTheFlag::
    .string "Have you decoded the flag yet?\n"
    .string "Check out CyberChef if you need help!$"
```

I also think that this change probably hurts the original game progression, as I think you can never get the Bike from the BikeShop. Maybe I'll fix that in the future.

---

# Encoding the Flag

The `FLAG` given by the Bike Shop is not in plaintext. Instead, it is XOR encoded using with the `FLAG_KEY`.

Decoding it is straightforward once both pieces are obtained. Players could use any tool they wanted, but the in-game dialogue even suggested using CyberChef if they needed help.

---

# Looking at the Patch

I've already explained most of the game mechanic changes in the above sections including:
- Modifying game events at map locations (Players House and BikeShop)
- Adding new dialogues
- Added new state variables (FLAG) to trigger alternate routes

Let's look at how items are added

## Adding New Items

Two new items were introduced:

- `FLAG_KEY`
- `FLAG`

The `FLAG_KEY` acts as the XOR key, while `FLAG` contains the encoded flag string.

These were added as standard key items so they appear in the player's inventory.

In `src/data/items.json`,

```json
{
    "english": "FLAG KEY",
    "itemId": "ITEM_FLAG_KEY",
    "price": 0,
    "holdEffect": "HOLD_EFFECT_NONE",
    "holdEffectParam": 0,
    "description_english": "catch-em-all",
    "importance": 1,
    "registrability": 1,
    "pocket": "POCKET_KEY_ITEMS",
    "type": "ITEM_TYPE_BAG_MENU",
    "fieldUseFunc": "FieldUseFunc_OakStopsYou",
    "battleUsage": 0,
    "battleUseFunc": "NULL",
    "secondaryId": 0
},
{
    "english": "FLAG",
    "itemId": "ITEM_FLAG",
    "price": 0,
    "holdEffect": "HOLD_EFFECT_NONE",
    "holdEffectParam": 0,
    "description_english": "1704150e455f0a0e460418411309\\n1d111b4848054c13410b0208",
    "importance": 1,
    "registrability": 1,
    "pocket": "POCKET_KEY_ITEMS",
    "type": "ITEM_TYPE_BAG_MENU",
    "fieldUseFunc": "FieldUseFunc_OakStopsYou",
    "battleUsage": 0,
    "battleUseFunc": "NULL",
    "secondaryId": 0
}
```

In `src/data/items.h`,
```c
{
    .name = _("FLAG KEY"),
    .itemId = ITEM_FLAG_KEY,
    .price = 0,
    .holdEffect = HOLD_EFFECT_NONE,
    .holdEffectParam = 0,
    .description = gItemDescription_ITEM_FLAG_KEY,
    .importance = 1,
    .registrability = 1,
    .pocket = POCKET_KEY_ITEMS,
    .type = ITEM_TYPE_BAG_MENU,
    .fieldUseFunc = FieldUseFunc_OakStopsYou,
    .battleUsage = 0,
    .battleUseFunc = NULL,
    .secondaryId = 0
}, {
    .name = _("FLAG"),
    .itemId = ITEM_FLAG,
    .price = 0,
    .holdEffect = HOLD_EFFECT_NONE,
    .holdEffectParam = 0,
    .description = gItemDescription_ITEM_FLAG,
    .importance = 1,
    .registrability = 1,
    .pocket = POCKET_KEY_ITEMS,
    .type = ITEM_TYPE_BAG_MENU,
    .fieldUseFunc = FieldUseFunc_OakStopsYou,
    .battleUsage = 0,
    .battleUseFunc = NULL,
    .secondaryId = 0
}
```

And in `include/constants/items.h`
```c
#define ITEM_FLAG_KEY 375
#define ITEM_FLAG 376

#define ITEMS_COUNT 377
```

---
# Solution

The solution was for players to:

1. Go to Professor Oak's lab  
2. Beat the rival  
3. Return home  
4. Receive the FLAG_KEY from mom  
5. Travel to Cerulean City Bike Shop
6. Obtain the encoded flag  
7. XOR decode it  

Since this is a Game Boy Advance ROM, players could use some tricks. This included things like:

- Patching the ROM to remove restrictions
- Using [cheats](https://www.pokémoncoders.com/pokémon-fire-red-gameshark-codes/)

One particularly easy shortcut was to teleport directly to the Cerulean Bike Shop using standard GBA cheat codes. This bypassed the gameplay entirely.

However, even if players skipped the game progression, they still needed to figure out:

- how to obtain the FLAG_KEY  
- how to decode the XOR encoded flag  

---

# Huge Shoutout to PRET

This challenge turned out to be a lot of fun to build. The [pret Pokémon project](https://pret.github.io/) made it really easy to modify the game. Without this project, implementing the challenge would have been far more difficult.

Compared to traditional binaries or web challenges, interacting with a full game definitely created a much more fun and engaging experience.

---

# Final Thoughts

If you ever wanted to hide a flag inside a video game, the Pokémon decompilation projects are a great place to start. The scripting system and event mechanics make it easy to modify the game and create really interesting challenges.

Watching competitors play through Pokémon FireRed was a lot of fun.