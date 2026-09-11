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
  - Start
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
  - Start
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
    - Start: T1
    - EndTime: T2
    - Participants
      - P1
      - P2
    - Dialogs
      - Dialog
        - ID: Dialog1
        - Start: T1
        - Duration: T2 - T1
        - Contributors
          - P1
          - P2
        - Participants
          - P1
          - P2
        - body: {}



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
    - Start: T1
    - EndTime: T2
    - Participants
      - Party1
      - Party2
      - Party3
    - Dialogs
      - Dialog
        - ID: Dialog1
        - Start: T1
        - Duration: T2 - T1
        - Contributors
          - Party1
          - Party2
          - Party3
        - body: {}
    - Events
      - Event
        - ID: 1
        - type: join
        - time: T1 + 1
        - participant: Party1
      - Event
        - ID: 2
        - type: join
        - time: T1 + 2
        - participant: Party2
      - Event
        - ID: 3
        - type: join
        - time: T1 + 4
        - participant: Party3


It looks a lot like the VCON for the 2-party call with notable differences. First, there is a third party in the party list. There is still a single dialog, representing the recording of the entire meeting from time T1 to T2. We also have the list of contributors - Party1, Party2 and Party3. Being in the list means that they contributed to the dialog at some point. Notice however, there is no list of contributors in the dialog object. This is because the three parties joined at different times a few minutes after the meeting start (1, 2 and 4mins into the meeting). The definition provided for the contributors object in a dialog is that - presence there means that user received the entirety of the dialog. In this case, none of them were present for the entirety.

There is a participant list in the session itself, which conveys the general list of "who was in the meeting" without consideration of exactly when those users were in the meeting.


## Online Meeting - Speaker Separation

The previous use case had a single, mixed audio recording for the entire meeting, containing the combination of the audio contributed by all three participants. In this example, the vcon instead provides speaker separated audio, with each participant having its own single-channel audio recording.

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
    - Start: T1
    - EndTime: T2
    - Participants
      - Party1
      - Party2
      - Party3
    - Dialogs
      - Dialog
        - ID: Dialog1
        - Start: T1 + 1
        - Duration: T2 - (T1 + 1)
        - Contributors
          - Party1
        - body: {}
      - Dialog
        - ID: Dialog2
        - Start: T1 + 2
        - Duration: T2 - (T1 + 2)
        - Contributors
          - Party2
        - body: {}
      - Dialog
        - ID: Dialog3
        - Start: T1 + 4
        - Duration: T2 - (T1 + 4)
        - Contributors
          - Party3
        - body: {}
    - Events
      (omitted for clarity)

The important thing to note in this case is that there are three dialogs, and they are for overlapping periods of time. Each dialog ran for the duration for which its respective participant was in the meeting. Furthermore, each dialog has but a single contributor - the one party whose audio is represented in that dialog.

This use case is really interesting, since it provides speaker separated audio, without making use of multiple channels in the audio format itself. Rather, it is the vcon which provides the speaker separation.


## Online Meeting - Speaker Separated Multi-Channel

In this use case, we once again have an online meeting. However, the audio recording format uses a multi-channel format, and each participant is present as one of the channels in the audio file.

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
    - Start: T1
    - EndTime: T2
    - Participants
      - Party1
      - Party2
      - Party3
    - Dialogs
      - Dialog
        - ID: Dialog1
        - Start: T1
        - Duration: T2 - T1
        - Contributors
          - Party1
          - Party2
          - Party3
        - ChannelParticipantMap
          - 0: Party1
          - 1: Party2
          - 2: Party3
        - body: {}
    - Events
      (omitted for clarity)


You can see in the example above there is a single dialog with a body. The body would be a multi-channel audio file. THere is now also an explicit ChannelParticipantMap object which maps the participants into the channels.

## AI Agent Voice Session

We now consider the interesting case of an AI Agent voice chat session. In this simple case, a user is speaking to a customer support voice AI Agent, which they have called over the phone. The vcon was produced by the AI Agent harness. Because this is where the recording is taken, the vcon is able to represent each turn of the conversation as a distinct dialog. We can see that in the increasing start timers for each dialog.

- Parties
    - Party
      - ID: Party1
      - Name: Alice
      - tel: +17325551234
      - tyoe: person
    - Party
      - ID: Party2
      - Name: Erica
      - tel: +18007329194
      - type: bot
- Sessions
  - Session
    - ID: Sess1
    - Start: T1
    - EndTime: T2
    - Participants
      - Party1
      - Party2
    - Dialogs
      - Dialog
        - ID: Turn1
        - Start: T1
        - Duration: EndOfTurn1 - T1
        - Contributors
          - Party2
        - body: {audio of AI agent initial greeting}
        - utteranceType: configured
      - Dialog
        - ID: Turn2
        - Start: StartOfTurn2
        - Duration: EndOfTurn2 - StartOfTurn2
        - Contributors
          - Party1
        - body: {audio of human response}
      - Dialog
        - ID: Turn3
        - Start: StartOfTurn3
        - Duration: EndOfTurn3 - StarOfTurn3
        - Contributors
          - Party2
        - body: {audio of AI agent first response}
        - tokens: 45342
        - utteranceType: llm-generated
      - Dialog
        - ID: Turn4
        - Start: EndOfTurn3 - 2s // (an interruption of 2s)
        - Duration: EndOfTurn4 - (EndOfTurn3 - 2s)
        - Contributors
          - Party1
        - body: {audio of human overtalk}
    - Events
      (omitted for clarity)
- Analysis
  - Analysis
    - type: transcript
    - dialogId: Turn1
    - transcript: "Hi thanks for calling Erica how can I help you"
  - Analaysis
    - type: transcript
    - dialogId: Turn2
    - transcript: "Yeah I am really mad about this late fee"
  - Analaysis
    - type: transcript
    - dialogId: Turn3
    - transcript: "I can help you with that, how about I"
  - Analaysis
    - type: transcript
    - dialogId: Turn4
    - transcript: "I don't want to hear it, put a human on"


There are several interesting things to note about this vcon.

First, the Party list shows two parties. The second of them is an AI Agent. It is designated as such, with a type of bot. We also see that the AI Agent has a name, in this case it is "Erica" which is the branded agent name used by Bank of America on their website and apps. Notice that there is also a phone number for the agent, which is the Bank of America number that the customer called to reach the agent.

You can also see from the vcon that each turn of the conversation is a unique dialog. This allows us to capture each utterance independently. That is useful because it also allows us to adorn each utterance with meta-data. You can see in the example above that there is a "tokens" attribute on one the agent dialogs which indicates how many tokens were spent (this is just examplary of the idea of how meta-data works). A second example of meta-data is also present in the example - the "utteranceType" attribute. This attribute indicates whether the audio content was created by an LLM inference operation (llm-generated) or is hard-coded by the agent designer (configured). Both of these examples illustrate why it is really a good idea to have each turn modeled as a distinct dialog, because we can include meta-data specific to that audio segment.

Another interesting element of this vcon is that there is overtalk. You can see that the human user interrupted the AI Agent at the end of its turn. This is a common occurence. The timestamps in the two dialogs (Turn3 and Turn4) clearly indicate what has happened. The user's interruption (Turn4 dialog) starts 2 seconds before the end of the AI Agents dialog (Turn3), indicating that there was an interruption and also conveying its exact duration.

The final interesting element of this vcon is the inclusion of an Analysis section, which here shows the transcript of each of the turns. Note that, each analysis entry refers to the dialog it analyzes using the dialogId. This replaces the mechanism in [VCON] which references a dialog by indiex. Because we can now have multiple sessions, each with its own set of dialogs, it is no longer possible to reference the dialog by index. Because the ID is always unique within the vcon, each analysis element can reference any dialog.



## AI Agent Chat Session

This example is actually quite similar to the one above, but now shows how it works when the exact same exchange happens over SMS, using the same phone numbers.

- Parties
    - Party
      - ID: Party1
      - Name: Alice
      - tel: +17325551234
      - tyoe: person
    - Party
      - ID: Party2
      - Name: Erica
      - tel: +18007329194
      - type: bot
- Sessions
  - Session
    - ID: Sess1
    - Start: T1
    - EndTime: T2
    - Participants
      - Party1
      - Party2
    - Dialogs
      - Dialog
        - ID: Turn1
        - Start: T1
        - Duration: EndOfTurn1 - T1
        - Contributors
          - Party2
        - body: "Hi thanks for calling Erica how can I help you"
        - utteranceType: configured
      - Dialog
        - ID: Turn2
        - Start: StartOfTurn2
        - Duration: EndOfTurn2 - StartOfTurn2
        - Contributors
          - Party1
        - body: "Yeah I am really mad about this late fee"
      - Dialog
        - ID: Turn3
        - Start: StartOfTurn3
        - Duration: EndOfTurn3 - StarOfTurn3
        - Contributors
          - Party2
        - body: "I can help you with that, how about I reduce it by 10%"
        - tokens: 45342
        - utteranceType: llm-generated
      - Dialog
        - ID: Turn4
        - Start: StartOfTurn4
        - Duration: EndOfTurn4 - StartOfTurn4
        - Contributors
          - Party1
        - body: "I don't want to hear it, put a human on"
    - Events
      (omitted for clarity)
- Analysis
  - Analaysis
    - type: sentiment
    - dialogId: Turn4
    - sentiment: angry


There are several differences in the resulting vcon to be noted.

Firstly, the body attribute of each dialog now contains the actual text, rather than an audio file. There is also no transcript element in the analysis, because it is clearly not needed - the media was natively text. THere is also no longer an interruption - something less meaningful in text-based interactions. However, the timestamps are still highly relevant even in the text interaction. They help convey how long the LLM took to generate the response; how long the user waiting before starting to type their answer; and how long it took them to type their answer.


## Coding Agent with SubAgents

In this example, we show one of the main use cases for multiple sessions in a vcon - usage of subagents. It is becoming more common for one AI Agent to spwan a second agent - synthesizing its prompt  - in order to take care of some kind of task. In this example, the user is interacting with a coding agent. They've asked the coding agent to explain a piece of code. To do so, it has spawned a sub-agent to analyze the code, and provide its results back.

- Parties
    - Party
      - ID: Party1
      - Name: Alice
    - Party
      - ID: Party2
      - Name: ClaudeCode
      - type: bot
    - Party
      - ID: Party3
      - Name: ClaudeCode SubAgent
      - type: bot
- Sessions
  - Session
    - ID: Sess1
    - Participants
      - Party1
      - Party2
    - Dialogs
      - Dialog
        - ID: Turn1
        - Contributors
          - Party1
        - body: "What does function ComputeRate do?"
      - Dialog
        - ID: Turn2
        - Contributors
          - Party2
        - body: "Spawning sub-agent to analyze it"
      - Dialog
        - ID: Turn3
        - Contributors
          - Party2
        - body: "It computes the interest via a database lookup of the user".
    - Events
      - Event
        - ID: 1
        - type: spawn-subagent
        - SpawnedSessionId: Sess2
      - Event
        - ID: 2
        - type: subagent-completion
        - SpawnedSessionId: Sess2
        - result: "It computes the interest via a database lookup of the user".
    - Session
      - ID: Sess2
      - Participants
        - Party2
        - Party3
      - Dialogs
        - Dialog
          - ID: Turn2.1
          - Contributors
            - Party2
          - body: "analyze the file RateComputer.java and summarize what it does."
        - Dialog
          - ID: Turn2.2
          - Contributors
            - Party3
          - body: "It computes the interest via a database lookup of the user".
      - Events
        - Event
          - ID: ToolCall1
          - type: ToolCallRequest
          - Contributors
            - Party2
          - tool-name: read file
        - Event
          - ID: ToolCall2
          - type: ToolCallResponse
          - Contributors
            - Party2
          - tool-name: read file
          - tool-result: {file}



This is now a much more complex case. For brevity, the timestamps have all been omitted.

Firstly, we see that there are two sessions. The main session is between Alice (Agent 1) and the Claude Code agent (Party 2). Alice asks for an analysis of a file. In that session, we see two important events - spawning of a sub-agent (which is a unique session) and completion of the subagent. Note that the spawning and completion events contain a reference to the session identifier for the session that was spawned.

The session that is spawned - is a proper session. It has participants. In this case, there are two of them - the main claude code agent, and the sub agent that it spawned. In this simplified example, the main agent provides input to the sub-agent - telling it to analyze a specific file - and the sub-agent comes back with its answer. This second session also has events. Here, there is a tool call request and a result. The tool call request is to read a file, and the response is the file itself. We also see that, in the second session, the sub-agent responds to the inquiry which its summary of what happnes in the file (dialog Turn2.2). This result is then taken by the main agent and passed on to the user in the main session (Turn 3).


## Meeting Sidebar

This final use case shows the usage of multiple sessions in a purely human conversation - the usage of sidebars in a meeting. In meetings, a sidebar is quite literally a second meeting with its own set of participants (potentially including new ones) and content. It is spawned from the main meeting, and when it ends, the participants return to the main meeting conversation. This is identical to what is happening in the AI subagent coding example above!

In this simple example, the main meeting has three participants, and it spaws a sidebar with two of them.

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
    - Participants
      - Party1
      - Party2
      - Party3
    - Dialogs
      - Dialog
        - ID: MainMtg
        - Contributors
          - Party1
          - Party2
          - Party3
        - body: {main mixed audio file}
    - Events
      - Event
        - ID: 1
        - type: start-sidebar
        - SpawnedSessionId: Sess2
      - Event
        - ID: 2
        - type: end-sidebar
        - SpawnedSessionId: Sess2
    - Session
      - ID: Sess2
      - Participants
        - Party2
        - Party3
      - Dialogs
        - Dialog
          - ID: Sidebar
          - Contributors
            - Party2
            - Party3
          - body: {sidebar mixed audio file}

In this example, there are two distinct audio recordings embedded in the VCON. One - for the main meeting. The other - for the sidebar. The structure of the VCON is the same as the coding sub-agent case. The second session is a child of the first; there are events for the spawn and completion operations, which also convey the ID of the spawned session.


# AI Authorship Declaration

No AI was used to author this document.


# Security Considerations {#security}

Not covered in this document.




--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
