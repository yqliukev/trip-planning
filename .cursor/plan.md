# Trip Planning - plan

The product is an **itinerary planning app**. Features revolve around aiding in trip planning and recording trip plans and displaying them with an UI. 

## Source of truth

| Question | Canonical doc |
|---|---|
| Project vision | This file |

## End Goal

A web application that can:

- Store travel information from users for long trips, including itinerary path, times, lodging, destinations, and other notes.
- Allow users to clearly access different levels of detail for a long trip, from broad descriptions like which days are in which region to a detailed itinerary for a single day.
- Display information clearly, with an interactive map.
- Itinerary information can be accessed by agents / LLMs for grounding.
- Calculate potential commute routes.
- Store backup plans and situational information.

## First Useful Version

- A polished UI which allows the user to store general notes about each trip location on a map

## Future Improvements

- Calculate the commute distance and time estimate between locations.
- More detailed layers: daily intineraries, backup plans.
- Section off common information types such as lodging, transit, and tickets.
- Provide suggestions for routes between multiple general areas.
- Way for agents / LLMs to access application data.