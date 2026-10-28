# Brave on-device speech recognition test page

This plan tests the three Web Speech API calls for on-device speech
recognition:

- `available()` asks if on-device speech works for a language.
- `install()` lets a website use the speech model. Brave asks the user before
  the first download.
- `start()` listens and returns text, from an audio track or the microphone.

Only English is supported. The plan does not test every option. It tests the
real model download, the real model, a real microphone and real websites.

## Before you start

- Use Brave Nightly 1.100.11 or later.
- **Test page**: open `https://yrliou.github.io/speech-test-page/`. Each
  button shows the code it runs. The log shows the result.
- **Profile**: use a brand-new profile. On macOS:

  ```
  "/Applications/Brave Browser Nightly.app/Contents/MacOS/Brave Browser Nightly" \
    --user-data-dir=/tmp/brave-speech-qa
  ```

  Windows and Linux take the same `--user-data-dir` option.
- **Feature flag**: `brave://flags/#brave-on-device-speech-recognition`. It is
  off by default. Section 2 turns it on.
- **Model**: 636 MB. Brave downloads it only after the user allows it.

Do the sections in order, on one profile. Section 6.2 needs a Brave Origin
build and a second new profile, with a different `--user-data-dir`.

## 1. Feature flag off

Delete `/tmp/brave-speech-qa`, then launch with the command above.

1. Open the test page. **Status** says `unavailable`.
2. In **2. start()**, keep **Sample clip**, tick **processLocally** and press
   **Start**. The log shows `error language-not-supported`.
3. Untick **processLocally** and press **Start**. The log shows an error, and
   no `interim` or `final` lines.

## 2. Feature flag on

1. Open `brave://flags/#brave-on-device-speech-recognition`. **Brave
   On-Device Speech Recognition** shows **Default**. Set it to **Enabled** and
   press **Relaunch**.
2. `brave://components` has no "Brave On-Device Speech Models" entry.
3. Reload the test page. **Status** says `downloadable`.
4. Tick **processLocally** and press **Start**. The log shows
   `error language-not-supported`.

## 3. install()

1. In **1. available() and install()**, press **install()**. A prompt opens:
   "yrliou.github.io wants to Download a speech recognition model (636 MB) to
   process speech on this device".
2. Press **Block**. The log shows `false`. `brave://components` has no model.
3. Press **install()** again, then press **Allow**. `brave://components`
   lists the model. The download can take a few minutes.
4. When the download is done, the log shows `true`. **Status** says
   `available`.
5. Set langs to `en-GB` and press **available()**. The log shows
   `"available"`.
6. Set langs to `fr-FR`. **available()** gives `"unavailable"`, and
   **install()** gives `false`.

## 4. start()

Use **Sample clip**, unless the step says **Microphone**. Reset the options to
the defaults after each row.

The sample clip plays through an audio track at the computer's audio rate.
The first log line shows it, usually 44100 or 48000 Hz. The model takes
16000 Hz, so this also tests the audio conversion.

### processLocally

| processLocally | Expected in the log |
| --- | --- |
| ticked | `interim` lines, then `final "This is a sentence in a single segment"` |
| unticked | The same |

The page stops listening 2 seconds after the clip ends. The `final` line comes
about 1 second after that.

### Options

Tick **processLocally** for each row.

| Option | Expected in the log |
| --- | --- |
| continuous unticked | Ends by itself about 1 second after the speech ends |
| interimResults unticked | Only the `final` line |
| lang `fr-FR` | `error language-not-supported` |

### Microphone

Pick **Microphone** and tick **processLocally**.

1. Press **Start** and allow the microphone. Say a sentence and press
   **Stop**. The `final` line shows your words.
2. Untick **continuous** and do it again. It stops by itself about 1 second
   after you stop talking.

### Real websites

These websites do not set processLocally. They show your words:

- `https://www.google.com/intl/en/chrome/demos/speech.html`: choose English,
  United States. Click the microphone, speak, click it again.
- `https://speechnotes.co/dictate/`: click the microphone, speak, click it
  again.

## 5. After a relaunch and on a new origin

1. Quit Brave and launch it again with the same command. Reload the test page.
   **Status** says `available`. Tick **processLocally** and press **Start**.
   The log shows the sentence.
2. Open `https://example.com`, which is a different origin, and the DevTools
   console. Typing in the console counts as a click. Run:
   - `await SpeechRecognition.available({ langs: ['en-US'], processLocally: true })`
     returns `"downloadable"`. Each origin must call `install()` once.
   - `await SpeechRecognition.install({ langs: ['en-US'], processLocally: true })`
     returns `true` right away, with no prompt.
   - `await SpeechRecognition.available({ langs: ['en-US'], processLocally: true })`
     returns `"available"`.

## 6. Other windows and builds

### 6.1 Private, Guest and Tor windows

This section needs the change that turns off on-device speech in these
windows. That change has not landed yet.

Use the same profile. The model is on disk, but these windows must not use
it. Open a **New Private Window**, a **Guest** window and a **New Private
Window with Tor**. In each, open the test page and check:

| Check | Expected |
| --- | --- |
| **Status** | `unavailable` |
| **install()** | `false`, with no prompt |
| **Start**, processLocally ticked | `error language-not-supported` |
| **Start**, processLocally unticked | `error network`, and no text |

### 6.2 Brave Origin builds

Origin builds have an **On-Device AI** switch at `brave://settings/origin`. It
starts off. Use a brand-new profile, and turn on the feature flag as in
section 2. Reload the test page after each change.

1. With the switch off:
   - **Status** says `unavailable`.
   - **Start** with processLocally ticked gives
     `error language-not-supported`.
   - **Start** with processLocally unticked gives an error, and no text.
2. Turn **On-Device AI** on. **Status** says `downloadable`. Press
   **install()**, then **Allow**. When the download is done, the log shows
   `true`. **Start** with processLocally ticked shows the sentence.
3. Turn **On-Device AI** off. The model disappears from `brave://components`.
   **Status** says `unavailable`. **Start** with processLocally unticked gives
   an error, and no text.

## Expected behavior

- With continuous ticked, the words stay `interim` until the page stops
  listening. Then one `final` arrives. Chrome does the same.
- Without processLocally, anything Brave does not support ends with an error.
  Brave has no speech server.

## Not in this plan

The plan does not test these items. Browser tests and unit tests in
brave-core cover most of them.

- The `quality` options.
- Esc on the prompt, and **Don't ask again**.
- The settings switch in `brave://settings/privacy`.
- `install()` 6 seconds after the click.
- The **WAV file** source.

## Credits

`wpt_speech.wav` is from web-platform-tests. See `wpt_speech.wav.LICENSE.md`.
