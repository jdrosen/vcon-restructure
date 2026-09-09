---
title: "Virtualized Conversations (VCON) Restructure to Facilitate AI Agent Use Cases"
abbrev: "VCON Restructure"
category: info

docname: draft-rosenberg-vcon-restructure-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Virtualized Conversations"
keyword:
 - next generation
 - unicorn
 - AI-native
venue:
  group: "Virtualized Conversations"
  type: "Working Group"
  mail: "vcon@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/vcon/"
  github: "jdrosen/vcon-restructure"
  latest: "https://jdrosen.github.io/vcon-restructure/draft-rosenberg-vcon-restructure.html"

author:
 -
    fullname: "Jonathan Rosenberg"
    organization: jdrosen.net
    email: "jdrosen@jdrosen.net"

normative:

informative:

  VCON: I-D.draft-ietf-vcon-vcon-core

  BIRK: I-D.draft-birkholz-verifiable-agent-conversations

  HOWE: I-D.draft-howe-vcon-agent-session-00



--- abstract

The Virtualized Conversations (VCON) specification provides a structured format for storing recordings of conversations, including phone calls, email threads and multi-party chats. VCONs also store metadata like call transcripts and mid-call events, like a call hold or addition of a party. Its structure is well suited for 2-party and basic multiparty phone calls. However, there is a need for the VCON format to also act as a record of AI Agent conversations, which are just another type of conversation. This document proposes changes to the object model in VCON to make it a more suitable format for handling AI Agent conversations, as well as more complex conferencing use cases. 


--- middle

# Introduction {#intro}

The Virtualized Conversations (VCON) specification [VCON] provides a structured format for storing recordings of conversations, including phone calls, email threads and multi-party chats. VCONs also store metadata like call transcripts and mid-call events, like a call hold or addition of a party. Its structure is well suited for 2-party and basic multiparty phone calls. 

However, there is a need for the VCON format to also act as a record of AI Agent conversations, which are just another type of conversation. Recent work by Birkholz [BIRK] has proposed a new format for recording AI Agent conversations, and a merged draft has been written to try and bring those into VCON [HOWE]. 

During the VCON interim meeting on 9th September 2026, a proposal was floated for a change to VCON that would cover these additional cases. This document writes up the change that was discussed. It presents the core data model, composed of two new objects - session and events. It then goes on to look at a variety of use cases and shows how they are handled by this data model.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Data Model

The basic data model is backwards compatible with the current vcon spec, and basically is composed of four primary objects - session (new), events (new), dialog (existing, but modified), and parties/participants (existing, but modified). VCON defines many other attributes, like extensions, subject, etc. which are unchanged and not discussed further here. The goal here is to explain just the primary first-class objects which are central to modeling use cases. Everything else is really just meta-data ontop of that.

## Session

The session object is new to this proposal. A session represents an instance of a continuous conversation amongst one or more participants, for which the context of the conversation starts empty and grows through the conversation. We use the term context here to refer to both the memory collected in the minds of the humans participating, but also to Large Language Model (LLM) context which would be fed into an LLM with an AI Agent in the conversation. A session has a start and it has an end. 

Within the confines of the session, there is a timeline that contains both events and dialogs. An event is a point-in-time action performed by a singular participant relevant to the conversation, such as pressing a DTMF key, the departure of a user from the conversation, or the invocation of a tool. A dialog is a record of information exchanged amongst participants, such as an audio recording, a text message, or an image. 

In a simple phone call between two people, there is a single session. A traditional conference call - like a Zoom or Webex meeting - would also be a single session. Even though participants may come and go in a meeting, there is still a collective advancement of the context as the meeting progresses. For example, a user might join 10 minutes late and then inquire, "hey, what happened so far?". The answer given by someone who has been in the meeting represents this context. 

Along similar lines, a conversation between a user and ChatGPT with fresh context represents a single session. If a user calls a 1-800 number and is connected to a voice AI Agent, that is also a single session. 

Importantly, a session can contain other sessions. This happens when the parent session spawns the child session. The child session - being a full-fledged session - has its own start and stop, and has a reset of context when it begins - though it will typically be started with context created by the parent session. We consider two examples to make this clear. The first is a meeting which spawns two sidebars. Each sidebar is, in its own right, a distinct meeting - with distinct participants, a distinct timeline. Similarly, an AI Agent session - such as a Claude Code session - can result in spawning of sub-agents to perform a certain task. Each of those would be a distinct session. 

A session has participants, which refer to the entities (humans and AI agents) which were privy to at least some portion of the session. They may not have all been there all of the time, however. 

A session has an ID, unique only to the internals of the VCON document. This allows for sessions to reference each other. 

The top level VCON has an array of sessions. One use case where there might be more than one, is when a VCON represents all of the calls made by a particular user in a given month, exported as a single VCON. In that case, there would be many sessions, one for each call. The top-level parties object would contain that user's party information, but also there would be an entry for every other person that the user in question spoke to. 

In summary:

- Session
  - Participants
  - StartTime
  - EndTime
  - ID
  - Child Sessions
  - Dialogs
  - Events

## Parties and Participants

The VCON specification defines Parties as an array of participants at the top-level of the VCON object. This remains as it is - a list of participants that were involved somehow in the conversaion(s) decribed by the VCON object. This document proposes adding an "id" parameter, which is an identifier for this participant unique only within the confines of the VCON object. This allows for easy reference to each party from the other objects in the data model - session, event and dialog.

A session contains an array of Participants, each one of which is the id of a participant that was involved in the session. As noted above, a party is included in the list of participants  n the session if that party was involved in the session at some point. 

A dialog contains an array of Participants, each of which is the id of a participant that received the entire content of the dialog.

An event contains a single Participant, which is the id of the participant that generated the event.

Simply put - Parties is a flag list of human or AI entities, without regard to their specific involvement in a session, dialog or event. Participants is a reference to one of more of those parties - referenced by id - indicating which of them were involved in a session, dialog or event. 

The current VCON includes a parties array as a child of dialog. The Participants array provides an alternative to it, referencing each participant specifically by ID, instead of by index. The usage of a reference by ID means it is trivial to redact/remove parties or participants without changing the indexing operation.

## Events

An event is a point-in-time action performed by a singular participant relevant to the conversation, such as pressing a DTMF key, the departure of a user from the conversation, or the invocation of a tool. 

An event always happens within a context of a session. In other words, the session is a container and it contains events (and dialogs). This means that the session object includes an array of events. This is optional, since there may not be any events. 

An event is always invoked by a particular participant, which is the one that is responsible for causing that event. An event, being a point-in-time, has a time at which it happened. Events have a type, which defines the type of event. And then it has data, which is a data specific to the event type. For a keyup event, the data would be the DTMF key that was pressed (e.g., the number 8). 

The event object replaces and subsumes the Party_History object defined in VCON. Each of the party history events -  join, drop, hold, unhold, mute, unmute, keydown, keyup - are a supported value for the event type. 

This document also suggests adding two additional events: start-speaking and end-speaking, which can be used to capture active speaker detection in a call or conference. These events would contain the id of the party who started (or stopped) speaking. 

For an AI Agent session, there would be additional events for tool call request, tool call response, and reasoning event, which would be defined in an extension to VCON for AI Agents. 

Transfer gets modeled as an event also - more accurately, by a set of events: blind-transfer, consult-start, consult-transfer, and consult-conference. The blind-transfer event is used for blind transfers. It contains a reference to the session (session, not dialog) which is the result of the blind transfer. Often this would be the last event in the parent session, since the session ends upon transfer. The consult-start event indicates that a consultation transfer has begun, and it too contains a reference to the session which represents the consultation call. If the consult call results in a transfer, the original session would see a consult-transfer, and the second spawned session ends. The consult-conference event handles the case where the consultation results in a conference and not a transfer. Examples are provided in detail in sections below.

The events are extensible. 

In summary - 

- event
    - id
    - time
    - participant
    - type
    - data

## Dialogs

A dialog represents a piece of content contributed by one or more participants, and then shared to one or more other participants in the session. A dialog has a start and optional duration. It also has a mediatype, allowing the dialog to represent audio, video, text, images or other content types. 

In this data model, Dialogs are included within a session. Each session has an array of Dialogs. The VCON spec allows the Dialog object to exist as a top-level object too. This document proposes retaining that for backwards compatibility; but go-forward implementations should include them only at the session level.

The main example of a dialog is an audio recording. Their usage in VCON allows for several distinct ways to represent the audio content of a session.

1. A single dialog for the entire session, representing the mix of audio from all participants, in a single audio channel.
2. A single dialog for the entire session, with multiple channels, and a channel dedicated to each participant
3. A single dialog for the entire session, with multiple channels, and each channel contains a mix of one or more participants [NOTE: This is a common use case when stereo recordings are made off of a multiparty call; one participant usually shows up in one channel, and the mix of the others in the second channel]
4. Multiple dialogs in the session, where each dialog represents a non-overlapping segment of time, and contains the mix of audio from all participants
5. Multiple dialogs in the session, where each dialog represents a non-overlapping segment of time, and contains the mix of audio from those participants who were speaking at that time,
6. Multiple dialogs in the session, where each dialog represents a non-overlapping segment of time, and contains the a multi-channel audio file where each participant that was speaking during that segment of time is in its own channel
7. Multiple dialogs in the session, where the dialogs are overlapping in time, and it contains the audio content from a single participant. For example, if there were three participants in a conference call, there would be three dialogs, each of which begins when that participant joins the meeting, and ends when that participant leaves

The goal is that the dialog object can support all of these, and it is up to the generator of the VCON to decide which to use. 

Dialogs are the core object in VCON and largely unchanged by this proposed restructuring. However, to make the role of participants clear, this document proposes adding the following additional parameters:

- Contributors: The list of parties (by id) who contributed to the content in this dialog.
- Participants: This is a list of parties (by id) who received this dialog in its entirety. 
- ChannelParticipantMap: This is a map which maps a channel in the audio recording, to a list of participants whose audio is mixed into that channel for the entirety of the dialog. If the audio recording is dual-channel, the map would have two entries - 0 and 1. This is optional and only relevant for multi-channel audio. 


Note well - the omission of a user from the participants list does NOT mean that they didn't receive that dialog. It just means there is no definitive statement about whether they received it or not. 

There is in essence two distinct lists of participants here - those that contributed, and those that received. This allows the dialog model to represent asymmetric use cases. One such example is a town-hall style meeting, where an executive is speaking to the company. In such a use case, there would be a single contributor, but multiple participants receiving it. Indeed, the model allows for the VCOn to capture who heard what portions of the meeting, which can be useful for cases where a user is required to participate in a meeting, and their attendance is to be recorded. 

For backwards compatibility, the dialog object still supports the parties attribute, but it is effectively superceded by the two new attributes defined here. 

The recording-set type is removed in this proposal, replaced by the session concept.

The transfer type is also removed in this proposal, replaced by transfer events. 

OPEN QUESTION: if a document is shared amongst participants, for example in a group chat - it was uploaded to the group chat - should that be a dialog and not an Attachment? 

In summary:

- Dialog
  - StartTime
  - Duration
  - ID
  - Contributors
  - Participants
  - ChannelParticipantMap


# Example Use Cases

This section shows example use cases and how the data model would be used to model it.

## Basic Phone Call

In this basic case, user A calls user B and they have a phone call. The call is recorded by their mobile operator or perhaps their UCaaS provider if it was a business call. 

The VCON in this case is simple - one session, one dialog. There are two parties, and both are contributors and participants in the mixed audio recording. The call started at time T1 and ended at time T2. The VCON would look like this.

- Parties
    - Party 
      - ID: Party1
      - Name: Alice
    - Party 
      - ID: Party2
      - Name: Bob
- Sessions
  - Session
    - ID: Sess1
    - StartTime: T1
    - EndTime: T2
    - Participants
      - P1
      - P2
    - Dialogs
      - Dialog
        - ID: Dialog1
        - StartTime: T1
        - Duration: T2 - T1
        - Contributors
          - P1
          - P2
        - Participants
          - P1
          - P2



## Basic Online Meeting

Consider a simple Webex or Zoom meeting or similar. There are three participants. They all join at slightly different times near the start of the meeting at T1. There is a single recording of the audio content of the meeting, which is a single channel audio recording containing a mix of all participants. 

- Parties
    - Party 
      - ID: Party1
      - Name: Alice
    - Party 
      - ID: Party2
      - Name: Bob
    - Party 
      - ID: Party3
      - Name: Charlie
- Sessions
  - Session
    - ID: Sess1
    - StartTime: T1
    - EndTime: T2
    - Participants
      - P1
      - P2
      - P3
    - Dialogs
      - Dialog
        - ID: Dialog1
        - StartTime: T1
        - Duration: T2 - T1
        - Contributors
          - P1
          - P2
          - P3
    - Events
      - Event
        - ID: 1
        - type: join
        - time: T1 + 1
        - participant: P1
      - Event
        - ID: 2
        - type: join
        - time: T1 + 2
        - participant: P2
      - Event
        - ID: 3
        - type: join
        - time: T1 + 4
        - participant: P3


It looks a lot like the VCON for the 2-party call with notable differences. First, there is a third party in the party list. There is still a single dialog, representing the recording of the entire meeting from T1 to T2. We also have the list of contributors - P1, P2 and P3. Being in the list means that they contributed to the dialog at some point. Notice however, the list of participants is removed from the dialog object. This is because the three participants joined at different times a few minutes after the meeting start. The definition provided for the participants object in a dialog is that - presence there means that user received the entirety of the dialog. In this case, none of them did because 




## Online Meeting - Speaker Separation


## AI Agent Voice Session


## AI Agent Chat Session


## SMS Conversation

## Contact Center - Queue Plus Agent


## Consultation Transfer



# AI Authorship Declaration

No AI was used to author this document. 


# Security Considerations {#security}

TODO Security


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
