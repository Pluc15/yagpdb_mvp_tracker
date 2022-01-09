# YAGPDB based MVP timer

* Copy paste the commands
* Update the `{{ /* Config */ }}` section

## Start a timer (!mvp)

Command to report a MvP death time

* `!mvp <mvp-name> <UtcTimeOfDeath:optional>`
* `!mvp thana 13h20`
* `!mvp thana`

```go
{{ /* Config */ }}
{{ $callbackCommandId := 2 }} {{ /* Set this to the ID of the Callback command */ }}
{{ $mentionRoleId := 929571707201663048 }} {{ /* Set this to the role ID for the people that wants tracker notifications */ }}

{{ /* Argument parsing */ }}
{{ $args := parseArgs
    1
    "!mvp <mvp-name> <UtcTimeOfDeath>"
    (carg "string" "MVP name")
    (carg "duration" "Time of death in UTC (eg 5h21)") }}

{{ $mvpDeathTime := (newDate currentTime.Year currentTime.Month currentTime.Day currentTime.Hour currentTime.Minute 0) }}
{{ if $args.IsSet 1 }}
    {{ $mvpDeathTime = (newDate currentTime.Year currentTime.Month currentTime.Day 0 0 0).Add ($args.Get 1) }}
{{ end }}
{{ $mvpKey := ($args.Get 0) }}

{{ /* Get MvP info from DB */ }}
{{ $mvpName := dbGet 118 (joinStr "" "mvp_" $mvpKey "_name") }}
{{ $mvpResp := dbGet 118 (joinStr "" "mvp_" $mvpKey "_resp") }}
{{ $mvpVar := dbGet 118 (joinStr "" "mvp_" $mvpKey "_var") }}
{{ $mvpWarn := dbGet 118 (joinStr "" "mvp_" $mvpKey "_warn") }}

{{ if $mvpName }}
    {{ /* Calculations */ }}
    {{ $mvpWarnAfter = sub $mvpResp $mvpWarn }}
    {{ $mvpWarnAfterDuration := (toDuration (mult $mvpWarnAfter .TimeMinute)) }}
    {{ $mvpWarnTime := ($mvpDeathTime.Add $mvpWarnAfterDuration) }}
    {{ $mvpWarnInSeconds := toInt (round ($mvpWarnTime.Sub currentTime).Seconds) }}
    {{ $mvpRespDuration := (toDuration (mult $mvpResp .TimeMinute)) }}
    {{ $mvpRespTime := ($mvpDeathTime.Add $mvpRespDuration) }}
    {{ $mvpRespInSeconds := toInt (round ($mvpRespTime.Sub currentTime).Seconds) }}

    {{ /* Save the new time in the DB */ }}
    {{ $mvpTimerKey := (joinStr "" "mvp_timer_" $mvpKey) }}
    {{ dbSet 118 $mvpTimerKey $mvpRespTime }}

    {{ /* Send message */ }}
    {{ if le $mvpRespInSeconds 0  }}
        {{ print $mvpName " already respawned in the respawn window" }}
    {{ else }}
        {{ print $mvpName " died <t:" $mvpDeathTime.Unix ":R>. It will spawn again <t:" $mvpRespTime.Unix ":R>" }}
    {{ end }}

    {{ /* Schedule the warning notification */ }}
    {{ if le $mvpWarnInSeconds 0  }}
        {{ print $mvpName " already in the notification window" }}
    {{ else }}
        {{ execCC $callbackCommandId nil $mvpWarnInSeconds (sdict "MvpName" $mvpName "RespawnTime" $mvpRespTime "MentionRoleId" $mentionRoleId) }}
    {{ end }}

    {{ /* Update Dashboard spawn warning message */ }}
    {{ $mvpDashbordMessageId := (dbGet 118 "mvp_dashbord_message_id").Value }}
    {{ if $mvpDashbordMessageId }}
        {{ $mvpTimers := dbGetPattern 118 "mvp_timer_%" 100 0 }}
        {{ $mvpDashboard := "__**MvP Dashboard**__" }}
        {{ range $mvpTimer := $mvpTimers }}
            {{ $mvpDashboard = joinStr "" $mvpDashboard "\n" $mvpTimer.Key ": <t:" $mvpTimer.Value.Unix ":R>" }}
        {{ end }}
        {{ editMessage nil $mvpDashbordMessageId $mvpDashboard }}
    {{ else }}
        {{ print "Dashboard not configured." }}
    {{ end }}
{{ else }}
    {{ print "Unknown MvP" }}
{{ end }}
```

## Callback for the timer warning (no trigger)

```go
{{ print .ExecData.MvpName " spawning <t:" .ExecData.RespawnTime.Unix ":R>" }}
{{ mentionRoleID .ExecData.MentionRoleId }}
```

## Configure dashboard channel (!mvpdashboard)

* `!mvpdashboard`

```go
{{ $messageId := sendMessageRetID nil "__**MvP Dashboard**__\nEmpty" }}
{{ dbSet 118 "mvp_dashbord_message_id" (str $messageId) }}
```

## Clear a timer (!mvpclear)

* `!mvpclear <mvp-key>`
* `!mvpclear thana`

WIP

## Init MVP database (!mvpinit)

Command you need to run on every MVP keys you want to be trackable with `!mvp`

* `!mvpinit <mvp-key>`
* `!mvpinit thana`

```go
{{ $args := parseArgs 1 "!mvpinit <mvp-key>"
    (carg "string" "Key of the MVP to init") }}

{{ $mvpNames := sdict
    "amonra" "Amon Ra (moc_pryd06)"
    "bio3" "Bio 3 (lhz_dun03)"
    "whitelady" "Bacsojin / White Lady (lou_dun03)"
    "bapho" "Baphomet (prt_maze03)"
    "darklord" "Dark Lord (gl_chyard)"
    "darklordgd" "Dark Lord (gld_dun04)"
    "detale" "Detale / Detardeurus (abyss_03)"
    "dopple" "Doppelganger (gef_dun02)"
    "dopplegd" "Doppelganger (gld_dun02)"
    "drac" "Dracula (gef_dun01)"
    "drake" "Drake (treasure02)"
    "eddga" "Eddga (pay_fild11)"
    "eddgagd" "Eddga (gld_dun01)"
    "esl" "Evil Snake Lord (gon_dun03)"
    "garm" "Garm / Hatii (xmas_fild01)"
    "gtb" "Golden Thief Bug (prt_sewb4)"
    "samurai" "Incantation Samurai / Samurai Specter (ama_dun03)"
    "kiel" "Kiel D-01 (kh_dun02)"
    "stormy" "Stormy Knight (xmas_dun02)"
    "ladytanee" "Lady Tanee (ayo_dun02)"
    "lod" "Lord of Death (niflheim)"
    "maya" "Maya (anthell02)"
    "mayagd" "Maya (gld_dun03)"
    "mistress" "Mistress (mjolnir_04)"
    "moonlight" "Moonlight Flower (pay_dun04)"
    "orchero14" "Orc Hero (gef_fild14)"
    "orchero02" "Orc Hero (gef_fild02)"
    "orclord" "Orc Lord (gef_fild10)"
    "osiris" "Osiris (moc_pryd04)"
    "pharaoh" "Pharaoh (in_sphinx5)"
    "phreeoni17" "Phreeoni (moc_fild17)"
    "phreeoni15" "Phreeoni (moc_fild15)"
    "rsx" "RSX 0806 (ein_dun02)"
    "taogunka" "Tao Gunka (beach_dun)"
    "thana" "Thanatos (thana_boss)"
    "turtle" "Turtle General (tur_dun04)"
    "valk" "Valkyrie Randgris (odin_tem03)"
    "vesper" "Vesper (jupe_core)"
    "bio2" "Ygnizem / Egnigem Cenia (lhz_dun02)"
 }}

{{ $mvpResps := sdict
    "amonra" 60
    "bio3" 100
    "whitelady" 117
    "bapho" 120
    "darklord" 60
    "darklordgd" 480
    "detale" 180
    "dopple" 120
    "dopplegd" 480
    "drac" 60
    "drake" 120
    "eddga" 120
    "eddgagd" 480
    "esl" 94
    "garm" 120
    "gtb" 60
    "samurai" 91
    "kiel" 120
    "stormy" 60
    "ladytanee" 420
    "lod" 133
    "maya" 120
    "mayagd" 480
    "mistress" 120
    "moonlight" 60
    "orchero14" 60
    "orchero02" 1440
    "orclord" 120
    "osiris" 60
    "pharaoh" 60
    "phreeoni17" 120
    "phreeoni15" 120
    "rsx" 125
    "taogunka" 300
    "thana" 120
    "turtle" 60
    "valk" 480
    "vesper" 120
    "bio2" 120 }}

{{ $mvpVars := sdict
    "amonra" 10
    "bio3" 30
    "whitelady" 10
    "bapho" 10
    "darklord" 10
    "darklordgd" 10
    "detale" 10
    "dopple" 10
    "dopplegd" 10
    "drac" 10
    "drake" 10
    "eddga" 10
    "eddgagd" 10
    "esl" 10
    "garm" 10
    "gtb" 10
    "samurai" 10
    "kiel" 60
    "stormy" 10
    "ladytanee" 10
    "lod" 0
    "maya" 10
    "mayagd" 10
    "mistress" 10
    "moonlight" 10
    "orchero14" 10
    "orchero02" 10
    "orclord" 10
    "osiris" 10
    "pharaoh" 10
    "phreeoni17" 10
    "phreeoni15" 10
    "rsx" 10
    "taogunka" 10
    "thana" 0
    "turtle" 10
    "valk" 10
    "vesper" 10
    "bio2" 10
 }}

{{ $mvpWarns := sdict
    "amonra" 5
    "bio3" 15
    "whitelady" 5
    "bapho" 5
    "darklord" 5
    "darklordgd" 5
    "detale" 5
    "dopple" 5
    "dopplegd" 5
    "drac" 5
    "drake" 5
    "eddga" 5
    "eddgagd" 5
    "esl" 5
    "garm" 5
    "gtb" 5
    "samurai" 5
    "kiel" 5
    "stormy" 5
    "ladytanee" 5
    "lod" 5
    "maya" 5
    "mayagd" 5
    "mistress" 5
    "moonlight" 5
    "orchero14" 5
    "orchero02" 5
    "orclord" 5
    "osiris" 5
    "pharaoh" 5
    "phreeoni17" 5
    "phreeoni15" 5
    "rsx" 5
    "taogunka" 5
    "thana" 5
    "turtle" 5
    "valk" 20
    "vesper" 15
    "bio2" 5
 }}

{{ $mvpKey := ($args.Get 0) }}

{{ dbSet 118 (joinStr "" "mvp_" $mvpKey "_name") ($mvpNames.Get $mvpKey) }}
{{ dbSet 118 (joinStr "" "mvp_" $mvpKey "_resp") ($mvpResps.Get $mvpKey) }}
{{ dbSet 118 (joinStr "" "mvp_" $mvpKey "_var") ($mvpVars.Get $mvpKey) }}
{{ dbSet 118 (joinStr "" "mvp_" $mvpKey "_warn") ($mvpWarns.Get $mvpKey) }}
{{ dbSet 118 (joinStr "" "mvp_key_" $mvpKey) ($mvpKey) }}
{{ dbSet 118 (joinStr "" "mvp_timer_" $mvpKey) ($mvpWarns.Get $mvpKey) }}

Initiated {{ $mvpNames.Get $mvpKey }}!
```

## Debug command (DB Dump)

```go
{{ $data := dbGetPattern 118 "mvp_%" 100 0 }}
{{ range $data }}
    {{ .Key }}: {{ .Value }}
{{ end }}
```