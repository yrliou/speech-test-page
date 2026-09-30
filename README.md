# Brave on-device speech recognition test page

Brave can now turn speech into text on the device, without sending audio to a
server. Websites use it through the Web Speech API. This plan checks the three
calls a website makes:

- `available()` asks whether on-device speech works for a language.
- `install()` lets a website use the speech model, downloading it first if
  needed.
- `start()` listens and returns text. It can listen to the microphone, or to
  an audio track the page plays.

Only English is supported for now.

## Before you start

- **Test page**: clone this repo, then in it run
  `python3 -m http.server 8765` and open `http://127.0.0.1:8765/`. Each button
  shows the code it runs, and the log at the bottom shows what came back.
- **Always launch with the feature flag.** There is no `brave://flags` entry,
  so use the command line. On macOS:

  ```
  "/Applications/Brave Browser Nightly.app/Contents/MacOS/Brave Browser Nightly" \
    --user-data-dir=/tmp/brave-speech-qa \
    --enable-features=BraveOnDeviceSpeechRecognition
  ```

  Windows and Linux take the same two flags. With the flag on, Brave
  downloads the speech model (about 600 MB) by itself at launch. Opening the
  profile without the flag deletes it, and it downloads again at the next
  launch with the flag.
- **Microphone only for now.** The page's **Sample clip** and **WAV file**
  sources are greyed out until Brave resamples audio tracks to the rate the
  model needs. Every **Start** below uses the microphone: press **Start**,
  allow the microphone the first time, say a sentence, then press **Stop**.

## 1. First install, on a brand-new profile

Delete `/tmp/brave-speech-qa` first, so the profile is new. Then launch with
the command above.

1. Open `brave://components`. "Brave On-Device Speech Models" starts at
   version 0.0.0.0 and shows a real version number once Brave has downloaded
   it, usually within a minute. No website has asked for it.
2. Open the test page. **Status** says `downloadable`, even though the model
   is there. Each website has to call `install()` once before it is told the
   model is available.
3. In **2. start()**, tick **processLocally** and press **Start**. The log
   shows `error language-not-supported`, because this website has not called
   `install()` yet.
4. Untick **processLocally**, press **Start**, say a sentence and press
   **Stop**. The log shows your words appearing in `interim` lines, then a
   `final` with your words. Without processLocally, `install()` is not
   needed.
5. In **1. available() and install()**, press **install()**. The log shows
   `true` right away, since the model is already downloaded, and **Status**
   changes to `available`.
6. Tick **processLocally** in **2. start()**, press **Start**, say a sentence
   and press **Stop**. The log shows a `final` with your words.
7. Quit Brave and launch it again with the same command. Reload the page.
   **Status** still says `available`.
8. Run `python3 -m http.server 8766` in the same folder and open
   `http://127.0.0.1:8766/`, which counts as a different website. **Status**
   says `downloadable`. Press **install()**. It returns `true` right away and
   **Status** changes to `available`.

## 2. Testing the calls with different options

Keep using the same profile and the page at `http://127.0.0.1:8765/`.

### available() and install()

Change the options in **1. available() and install()**, then press the
buttons. Put the options back to en-US, processLocally ticked and quality
(omit) after each row.

| Options | available() | install() |
| --- | --- | --- |
| en-US, processLocally (the defaults) | `available` | `true` |
| quality `dictation` | `available` | |
| langs `en-GB` | `available` | |
| quality `conversation` | `unavailable` | `false` |
| langs `fr-FR` | `unavailable` | `false` |
| processLocally unticked | `available` | `false` |

Without processLocally the browser always answers `available`, because it
assumes a server could do the work. Brave has no speech server, and the
**start()** tests below show what happens then.

### start()

Change the options in **2. start()**, press **Start**, say a sentence and
press **Stop**. Put the options back to the defaults after each row.

| Options | Expected in the log |
| --- | --- |
| The defaults | `interim` lines while you speak, then a `final` with your words after you press **Stop** |
| continuous unticked | Ends by itself about a second after you stop talking, with a `final` |
| interimResults unticked | Only the `final` line, no `interim` lines |
| processLocally ticked | `final` with your words |
| processLocally ticked, quality `dictation` | `final` with your words |
| lang `en-GB` | `final` with your words |
| processLocally ticked, quality `conversation` | `error language-not-supported` right away |
| quality `conversation` | `error network`, no text |
| processLocally ticked, lang `fr-FR` | `error language-not-supported` right away |
| lang `fr-FR` | `error network`, no text |

### Real websites

These call `start()` on the microphone too. They should show your words:

- `https://www.google.com/intl/en/chrome/demos/speech.html`: choose English,
  United States, click the microphone, speak, click it again.
- `https://speechnotes.co/dictate/`: click the microphone, speak, click it
  again.

### install() needs a click

Websites can only download the model in response to a click. Test this from
the DevTools console on a website you have not used for this test, for
example `https://example.com`. Typing in the console counts as a click.

1. `await SpeechRecognition.available({ langs: ['en-US'], processLocally: true })`
   returns `"downloadable"`.
2. `setTimeout(() => SpeechRecognition.install({ langs: ['en-US'], processLocally: true }).then(console.log, (e) => console.log(e.name)), 6000)`
   prints `NotAllowedError` after 6 seconds. The click has expired by then.
3. `await SpeechRecognition.install({ langs: ['en-US'], processLocally: true })`
   returns `true`.
4. `await SpeechRecognition.available({ langs: ['en-US'], processLocally: true })`
   returns `"available"`.

### On-Device AI switch (Brave Origin builds)

Brave Origin builds have an "On-Device AI" switch at `brave://settings/origin`,
and it starts off. Launch the Origin build with the same flag on a brand-new
profile. The switch takes effect right away, with no restart.

1. With the switch off, open the test page. **Status** says `unavailable`.
   **available()** gives `unavailable`, **install()** gives `false`, and
   **Start** gives `error language-not-supported` with processLocally ticked
   and `error network` without it. `brave://components` has no "Brave
   On-Device Speech Models" entry. The DevTools console on any website gives
   the same answers: `available()` returns `"unavailable"`, `install()`
   returns `false`.
2. Turn **On-Device AI** on. Brave downloads the model by itself, and
   `brave://components` lists it with a version number, usually within a
   minute. Reload the test page. **Status** says `downloadable`. Press
   **install()**: `true`, and **Status** changes to `available`. **Start**
   with processLocally ticked shows your words.
3. Turn **On-Device AI** off. Brave deletes the model right away, and the
   entry disappears from `brave://components`. Reload the test page.
   **Status** says `unavailable`.
4. Turn **On-Device AI** back on. The model downloads again. Once
   `brave://components` lists it, reload the test page. **Status** says
   `available` without pressing **install()** again, and **Start** with
   processLocally ticked shows your words.

## Expected behavior

- With continuous ticked, the words stay `interim` until you press **Stop**,
  and then one `final` arrives. Chrome does the same.
- Anything Brave does not support ends with `error network` unless
  processLocally is ticked, because Brave has no speech server.

## Credits

`wpt_speech.wav` is from web-platform-tests. See `wpt_speech.wav.LICENSE.md`.
