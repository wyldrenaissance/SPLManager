# Shared fake data for the five Dashboard drafts

Spaces
- Atrium: zones Lobby, Mezzanine, Gallery; computers RNDR-01, RNDR-02, RNDR-03
- Terrace: zones Terrace Deck, Pool Bar, Cabanas; computer RNDR-04

Zones (status / rendering / now playing / level / computers)
- Lobby        online   rendering  "Morning Drift" (Cloud)   -12 dB   RNDR-01
- Mezzanine    online   rendering  "Ambient Set 04" (Local)  -18 dB   RNDR-01, RNDR-02
- Gallery      degraded rendering  "Field Recordings" (Local) -9 dB   RNDR-02, RNDR-03
- Terrace Deck online   rendering  "Evening Tide" (Cloud)    -15 dB   RNDR-04
- Pool Bar     online   idle       (nothing)                 muted    RNDR-04
- Cabanas      offline  -          -                         -        RNDR-04

Computers (space / status / cpu / zones / note)
- RNDR-01 Atrium  online   34%  Lobby, Mezzanine
- RNDR-02 Atrium  online   41%  Mezzanine, Gallery
- RNDR-03 Atrium  degraded 88%  Gallery          "Audio interface dropouts"
- RNDR-04 Terrace online   22%  Terrace Deck, Pool Bar, Cabanas

Network: multicast 239.1.2.3, REST 7378, OSC 9002
Alerts: RNDR-03 audio interface dropouts (warn), Cabanas offline (off)
Tabs: Dashboard, Zones, Computers, Network, Speakermaps, Account, Settings
