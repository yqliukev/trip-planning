# Trip Planning - plan

The product is an **itinerary planning app**. Features revolve around aiding in trip planning and recording trip plans and displaying them with an UI. 

## Source of truth

| Question | Canonical doc |
|---|---|
| Project vision | This file |
| Architectural Decisions | [architecture file](architecture.md) |

## End Goal

A web application that helps the user create and manage a travel itinerary. The user builds a plan for a trip, revises it as details change, and reads it back from a broad overview down to a single day.

The application:

- Stores the itinerary for long trips — path, times, lodging, destinations, and other notes — and lets the user create, edit, and remove that information.
- Lets the user move between levels of detail, from which days fall in which region to a detailed itinerary for a single day.
- Displays the itinerary clearly, with an interactive map.
- Exposes itinerary information to agents / LLMs for grounding.
- Calculates potential commute routes.
- Stores backup plans and situational information.

### Agent

The agent expects a set of inputs from the user, such as:
- location(s)
- desired destinations / preferred scenery types
- budget

With those inputs, the agent uses the application in place of the user to create an itinerary. It drives the same actions a person would take in the app:
- Creating, combining, editing, and deleting trips
- Calculating and picking routes

The itinerary it produces is the one stored in the application.
