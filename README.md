# Open-Podcast

Open-Podcast is a Python project that leverages the Kokoro library to generate audio podcasts from a script. It provides a flexible framework to define different speakers, their voices, and generate natural-sounding audio segments with appropriate pauses.

## Features

*   **Script Parsing:** Parses a script with speaker designations, allowing for clear definition of dialogues.
*   **Speaker Configuration:** Allows defining unique voice settings for each speaker, including voice type, language code, and speech speed.
*   **Natural Audio Generation:** Utilizes the Kokoro library to generate realistic audio segments for each speaker.
*   **Customizable Pauses:** Adds pauses between segments and speaker changes to simulate natural conversation flow.
*   **Output to WAV:** Saves the generated podcast as a standard WAV audio file.
*   **Error Handling:** Implemented with try-except blocks, ensuring smooth continuation of podcast generation even if a segment fails.

## Prerequisites

Before using Open-Podcast, make sure you have the following installed:

*   **Python 3.6 or higher**
*   **pip** (Python package installer)

## Installation

1.  **Install Kokoro and Soundfile:**

    ```bash
    !pip install -q kokoro soundfile
    ```

2.  **Install espeak (optional but recommended):**

    espeak is used for out-of-dictionary fallback.  If you skip this step, words not found in the dictionary will be skipped.

    ```bash
    !apt-get -qq -y install espeak-ng > /dev/null 2>&1
    ```

## Usage

1.  **Import the necessary modules and the `PodcastGenerator` class:**

    ```python
    import re
    import numpy as np
    import soundfile as sf
    import random
    from kokoro import KPipeline
    ```

2.  **Define your podcast script:**

    The script should follow a simple format where each line starts with the speaker's name followed by a colon, and then the dialogue. Multi-line dialogue is supported.

    ```python
    script = """
    Host:
    Hello and welcome to the podcast!

    Ramanujan:
    It is a pleasure to be here.
    """
    ```

3.  **Create an instance of the `PodcastGenerator` class:**

    ```python
    generator = PodcastGenerator()
    ```

4.  **(Optional) Customize Speaker Settings:**
    Modify `self.speaker_settings` dictionary inside the `PodcastGenerator` class to configure specific voices and languages.
    ```python
        self.speaker_settings = {
            'Host': {'voice': 'af_heart', 'lang': 'a', 'speed': 1.0},
            'Ramanujan': {'voice': 'bm_daniel', 'lang': 'b', 'speed': 0.9}
        }
    ```

5.  **Generate the podcast audio:**

    ```python
    output_file = generator.generate_podcast(script, output_file='my_podcast.wav')
    ```

    This will create a WAV file named `my_podcast.wav` containing the audio generated from your script.

## Example

```python
import re
import numpy as np
import soundfile as sf
import random
from kokoro import KPipeline

class PodcastGenerator:
    def __init__(self, sample_rate=24000):
        self.sample_rate = sample_rate
        self.pipeline_cache = {}
        # Added a 'speed' key for each speaker.
        self.speaker_settings = {
            'Host': {'voice': 'af_heart', 'lang': 'a', 'speed': 1.0},
            'Ramanujan': {'voice': 'bm_daniel', 'lang': 'b', 'speed': 0.9}
        }

    def parse_script(self, script):
        """
        Improved script parsing that correctly handles:
        1. Speaker names appearing within dialogue
        2. Multi-line dialogue blocks
        3. Proper line continuation
        """
        segments = []
        current_speaker = None
        current_text = []

        # Split the script into lines and process
        lines = [line.strip() for line in script.strip().split('\n')]
        i = 0

        while i < len(lines):
            line = lines[i]
            if not line:  # Skip empty lines
                i += 1
                continue

            # Check if this line starts a new speaker block.
            # A new speaker block must:
            # 1. Start at the beginning of a line
            # 2. Have a colon
            # 3. Not be part of the previous dialogue
            if ':' in line and (i == 0 or not lines[i-1].strip()):
                parts = line.split(':', 1)
                potential_speaker = parts[0].strip()

                # Verify this is actually a speaker change
                if potential_speaker in self.speaker_settings:
                    # Save previous speaker's text if exists
                    if current_speaker and current_text:
                        segments.append((current_speaker, ' '.join(current_text)))
                        current_text = []

                    current_speaker = potential_speaker
                    if len(parts) > 1 and parts[1].strip():
                        current_text.append(parts[1].strip())
                else:
                    # Not a real speaker change, just part of dialogue
                    if current_speaker:
                        current_text.append(line)
            else:
                # Continuation of current speaker's text
                if current_speaker and line:
                    current_text.append(line)

            i += 1

        # Add the last speaker's text
        if current_speaker and current_text:
            segments.append((current_speaker, ' '.join(current_text)))

        return segments

    def get_pipeline(self, lang_code):
        """Get or create a cached KPipeline instance"""
        if lang_code not in self.pipeline_cache:
            self.pipeline_cache[lang_code] = KPipeline(lang_code=lang_code)
        return self.pipeline_cache[lang_code]

    def generate_audio_segment(self, text, voice, lang_code, speed=1.0):
        """
        Generate audio with optimized processing using kokoro's KPipeline.
        To produce a more natural-sounding delivery, the text is now split
        on both newlines and common punctuation marks (commas, periods, exclamation
        points, and question marks) so that brief pauses are inserted in places
        where a human speaker might naturally breathe or inflect.
        """
        pipeline = self.get_pipeline(lang_code)
        # Use a split pattern that captures both newlines and punctuation-based breaks.
        split_pattern = r'(?:\n+|(?<=[.!?])\s+)'
        generator = pipeline(text, voice=voice, speed=speed, split_pattern=split_pattern)
        # Concatenate all audio parts if available
        return np.concatenate([audio for _, _, audio in generator]) if generator else np.array([])

    def add_pause(self, base_duration_sec):
        """
        Generate a pause of base_duration_sec with a slight random variation
        to simulate natural breathing or turn-taking pauses.
        """
        # Introduce a random variation of ±0.1 sec
        variation = random.uniform(-0.1, 0.1)
        duration_sec = max(0, base_duration_sec + variation)
        num_samples = int(self.sample_rate * duration_sec)
        return np.zeros(num_samples, dtype=np.float32)

    def generate_podcast(self, script, output_file='podcast.wav'):
        """Generate podcast with improved script parsing and natural pauses"""
        segments = self.parse_script(script)
        audio_segments = []
        previous_speaker = None

        # Debug print to verify parsed segments
        print("\nParsed segments:")
        for speaker, text in segments:
            preview = text[:100] + "..." if len(text) > 100 else text
            print(f"\n{speaker}:\n{preview}")

        for speaker, text in segments:
            if speaker not in self.speaker_settings:
                continue

            settings = self.speaker_settings[speaker]
            print(f"\nGenerating {speaker}'s audio...", end='', flush=True)
            # Use the custom speed from speaker settings
            audio = self.generate_audio_segment(text, settings['voice'], settings['lang'], speed=settings.get('speed', 1.0))
            audio_segments.append(audio)

            # Decide on pause duration:
            # Longer pause when the speaker changes; shorter if continuing.
            if previous_speaker is None or previous_speaker != speaker:
                base_pause = 0.7  # Longer pause for speaker changes
            else:
                base_pause = 0.4  # Shorter pause if same speaker continues

            pause = self.add_pause(base_pause)
            audio_segments.append(pause)
            previous_speaker = speaker
            print("✓")

        if audio_segments:
            final_audio = np.concatenate(audio_segments)
        else:
            final_audio = np.array([], dtype=np.float32)

        sf.write(output_file, final_audio, self.sample_rate)
        print(f"\nPodcast successfully saved to {output_file}")
        return output_file

if __name__ == "__main__":
    script = """
    Host:
    Hello and welcome to the podcast!

    Ramanujan:
    It is a pleasure to be here.
    """

    generator = PodcastGenerator()
    output_file = generator.generate_podcast(script)

Customization

Speaker voices: The speaker_settings dictionary in the PodcastGenerator class allows you to specify the voice and language code for each speaker. Refer to the Kokoro documentation for a list of available voices and language codes.

Pause duration: You can adjust the base_pause variable in the generate_podcast method to control the length of pauses between audio segments.

Sample rate: You can set the sample rate when initializing the PodcastGenerator class. A higher sample rate will result in better audio quality but larger file sizes.

Splitting pattern You can customize split_pattern to split your text for more natural sound.

Error Handling

The generate_audio_segment function includes error handling to gracefully manage issues that may arise during audio generation, such as voice generation failures. If audio generation fails for a particular segment, it catches the exception, logs an error message, and returns an empty NumPy array, ensuring the podcast generation process can proceed without interruption.

Notes

The sample code assumes you are running it in a Colab or similar environment that allows shell commands (e.g., for installing espeak-ng). If running in a local environment, you may need to adjust the installation steps accordingly.
