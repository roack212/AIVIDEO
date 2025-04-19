import streamlit as st
import os
import tempfile
import shutil
import requests
import json
from elevenlabs import generate, set_api_key
import whisper
from moviepy.editor import VideoFileClip, AudioFileClip, CompositeVideoClip
from moviepy.video.tools.subtitles import SubtitlesClip
from moviepy.video.io.ffmpeg_tools import ffmpeg_extract_subclip
from gtts import gTTS
from pexels_api import API
import uuid

# ===== SET API KEYS (Using Environment Variables) =====
ELEVENLABS_API_KEY = os.getenv("ELEVENLABS_API_KEY", "your_elevenlabs_key")
PEXELS_API_KEY = os.getenv("PEXELS_API_KEY", "your_pexels_key")
set_api_key(ELEVENLABS_API_KEY)

# ===== 1. TEXT-TO-SPEECH (ELEVENLABS OR gTTS FALLBACK) =====
def text_to_speech(text, voice="Bella", use_elevenlabs=True):
    try:
        if use_elevenlabs and ELEVENLABS_API_KEY != "your_elevenlabs_key":
            audio = generate(text=text, voice=voice)
            output_path = os.path.join(tempfile.gettempdir(), f"voiceover_{uuid.uuid4()}.mp3")
            with open(output_path, "wb") as f:
                f.write(audio)
            return output_path
        else:
            # Fallback to gTTS (Google Text-to-Speech, copyright-free)
            tts = gTTS(text=text, lang='en')
            output_path = os.path.join(tempfile.gettempdir(), f"voiceover_{uuid.uuid4()}.mp3")
            tts.save(output_path)
            return output_path
    except Exception as e:
        st.error(f"Text-to-speech failed: {str(e)}")
        return None

# ===== 2. GENERATE SUBTITLES (WHISPER) =====
@st.cache_resource
def load_whisper_model():
    return whisper.load_model("base")

def generate_subtitles(audio_path):
    try:
        model = load_whisper_model()
        result = model.transcribe(audio_path)
        output_path = os.path.join(tempfile.gettempdir(), f"subtitles_{uuid.uuid4()}.srt")
        with open(output_path, "w") as f:
            for i, segment in enumerate(result["segments"]):
                start = segment['start']
                end = segment['end']
                f.write(f"{i+1}\n{int(start//60):02d}:{int(start%60):02d}.{int(start*1000%1000):03d} --> "
                        f"{int(end//60):02d}:{int(end%60):02d}.{int(end*1000%1000):03d}\n{segment['text']}\n\n")
        return output_path
    except Exception as e:
        st.error(f"Subtitle generation failed: {str(e)}")
        return None

# ===== 3. GET STOCK VIDEO (PEXELS) =====
def get_stock_video(query="nature", orientation="landscape"):
    try:
        api = API(PEXELS_API_KEY)
        api.search_videos(query, page=1, results_per_page=5, orientation=orientation)
        videos = api.get_videos()
        if not videos:
            raise ValueError("No videos found for the query.")
        video_url = videos[0].video_files[0].link
        output_path = os.path.join(tempfile.gettempdir(), f"stock_video_{uuid.uuid4()}.mp4")
        response = requests.get(video_url, stream=True)
        with open(output_path, "wb") as f:
            for chunk in response.iter_content(chunk_size=8192):
                if chunk:
                    f.write(chunk)
        return output_path
    except Exception as e:
        st.error(f"Failed to fetch stock video: {str(e)}")
        return None

# ===== 4. ADD BACKGROUND MUSIC (ROYALTY-FREE) =====
def add_background_music(video_path, duration, music_url="https://www.bensound.com/bensound-music/bensound-summer.mp3"):
    try:
        music_path = os.path.join(tempfile.gettempdir(), f"bg_music_{uuid.uuid4()}.mp3")
        response = requests.get(music_url)
        with open(music_path, "wb") as f:
            f.write(response.content)
        video = VideoFileClip(video_path)
        music = AudioFileClip(music_path).subclip(0, min(duration, video.duration))
        final_audio = CompositeAudioClip([video.audio, music.volumex(0.3)])  # Lower music volume
        final = video.set_audio(final_audio)
        output_path = os.path.join(tempfile.gettempdir(), f"video_with_music_{uuid.uuid4()}.mp4")
        final.write_videofile(output_path, codec="libx264", audio_codec="aac")
        return output_path
    except Exception as e:
        st.error(f"Failed to add background music: {str(e)}")
        return video_path  # Return original video if music fails

# ===== 5. COMBINE EVERYTHING =====
def create_video(voiceover, subtitles, stock_video, video_type="reel", subtitle_style="white"):
    try:
        temp_dir = tempfile.mkdtemp()
        video = VideoFileClip(stock_video)
        audio = AudioFileClip(voiceover)
        duration = audio.duration
        if video_type == "reel":
            duration = min(duration, 60)  # Max 60 seconds for reels
            video = video.resize((1080, 1920))  # 9:16 aspect ratio
        else:
            duration = min(duration, 300)  # Max 5 minutes for long videos
            video = video.resize((1920, 1080))  # 16:9 aspect ratio
        video = video.subclip(0, duration).set_audio(audio.subclip(0, duration))
        
        # Add subtitles
        def make_textclip(txt):
            return TextClip(txt, fontsize=40, color=subtitle_style, font="Arial", bg_color="black", size=(video.w, 100))
        subtitle_clip = SubtitlesClip(subtitles, make_textclip)
        subtitle_clip = subtitle_clip.set_pos(('center', 'bottom')).set_duration(duration)
        final = CompositeVideoClip([video, subtitle_clip])
        
        output_path = os.path.join(temp_dir, f"final_output_{uuid.uuid4()}.mp4")
        final.write_videofile(output_path, codec="libx264", audio_codec="aac")
        return output_path
    except Exception as e:
        st.error(f"Video creation failed: {str(e)}")
        return None
    finally:
        shutil.rmtree(temp_dir, ignore_errors=True)

# ===== STREAMLIT UI =====
def main():
    st.title("🎥 AI Text-to-Video Generator")
    st.markdown("Convert your text into videos with copyright-free voiceovers, subtitles, and background music! "
                "Create short reels or long videos.")

    # User Inputs
    st.subheader("Step 1: Enter Your Script")
    user_text = st.text_area("Your script (max 500 words for reels, 2000 for long videos):", 
                             "Hello, welcome to my AI-generated video! This is a sample script.")
    word_count = len(user_text.split())
    if word_count > 500 and st.session_state.get("video_type", "reel") == "reel":
        st.warning("Script too long for reels (max 500 words).")
    elif word_count > 2000:
        st.warning("Script too long for long videos (max 2000 words).")

    st.subheader("Step 2: Customize Your Video")
    video_type = st.selectbox("Video Type", ["Reel (Short, 9:16)", "Long Video (16:9)"], 
                              on_change=lambda: st.session_state.update({"video_type": "reel" if video_type == "Reel (Short, 9:16)" else "long"}))
    voice_option = st.selectbox("Voice", ["Bella (ElevenLabs)", "Rachel (ElevenLabs)", "Antoni (ElevenLabs)", "gTTS (Free)"])
    video_theme = st.text_input("Video Theme (e.g., nature, city, technology):", "nature")
    subtitle_style = st.selectbox("Subtitle Color", ["White", "Yellow", "Blue"])
    music_option = st.checkbox("Add Royalty-Free Background Music", value=True)
    
    if st.button("Generate Video"):
        with st.spinner("Generating your video..."):
            progress = st.progress(0)
            
            # Step 1: Generate Voiceover
            use_elevenlabs = voice_option != "gTTS (Free)"
            voice = voice_option.split()[0] if use_elevenlabs else None
            voiceover = text_to_speech(user_text, voice, use_elevenlabs)
            if not voiceover:
                st.stop()
            progress.progress(25)
            
            # Step 2: Generate Subtitles
            subtitles = generate_subtitles(voiceover)
            if not subtitles:
                st.stop()
            progress.progress(50)
            
            # Step 3: Get Stock Video
            orientation = "portrait" if video_type == "reel" else "landscape"
            stock_video = get_stock_video(video_theme, orientation)
            if not stock_video:
                st.stop()
            progress.progress(75)
            
            # Step 4: Add Background Music (Optional)
            if music_option:
                stock_video = add_background_music(stock_video, duration=60 if video_type == "reel" else 300)
            
            # Step 5: Create Final Video
            final_video = create_video(voiceover, subtitles, stock_video, video_type=video_type.split()[0].lower(), subtitle_style=subtitle_style.lower())
            progress.progress(100)
            
            if final_video:
                st.success("Video generated successfully!")
                st.video(final_video)
                with open(final_video, "rb") as f:
                    st.download_button("Download Video", f, file_name=f"ai_video_{video_type.split()[0].lower()}.mp4")
            else:
                st.error("Failed to generate video. Please check the logs or try again.")

if __name__ == "__main__":
    main()
