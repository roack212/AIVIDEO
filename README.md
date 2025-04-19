import streamlit as st
import os
import tempfile
import shutil
from gtts import gTTS
import whisper
from moviepy.editor import VideoFileClip, AudioFileClip, CompositeVideoClip, TextClip
from moviepy.video.tools.subtitles import SubtitlesClip
from moviepy.video.io.VideoFileClip import VideoFileClip
from moviepy.audio.AudioClip import CompositeAudioClip
import uuid

# ===== 1. TEXT-TO-SPEECH (gTTS ONLY - FREE) =====
def text_to_speech(text):
    try:
        tts = gTTS(text=text, lang='en')
        output_path = os.path.join(tempfile.gettempdir(), f"voiceover_{uuid.uuid4()}.mp3")
        tts.save(output_path)
        return output_path
    except Exception as e:
        st.error(f"Text-to-speech failed: {str(e)}")
        return None

# ===== 2. GENERATE SUBTITLES (WHISPER - OFFLINE) =====
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

# ===== 3. USE LOCAL VIDEO/IMAGE (NO PEXELS API) =====
def get_local_video(video_type="reel"):
    try:
        # Replace this path with your own local video/image file
        placeholder_path = "placeholder.mp4"  # or "placeholder.jpg"
        
        if not os.path.exists(placeholder_path):
            # Fallback: Create a blank colored clip
            from moviepy.editor import ColorClip
            duration = 5  # seconds
            if video_type == "reel":
                clip = ColorClip((1080, 1920), color=(0, 100, 200), duration=duration)
            else:
                clip = ColorClip((1920, 1080), color=(100, 0, 200), duration=duration)
            output_path = os.path.join(tempfile.gettempdir(), f"blank_video_{uuid.uuid4()}.mp4")
            clip.write_videofile(output_path, fps=24)
            return output_path
        return placeholder_path
    except Exception as e:
        st.error(f"Failed to load local video: {str(e)}")
        return None

# ===== 4. COMBINE EVERYTHING (NO BACKGROUND MUSIC) =====
def create_video(voiceover, subtitles, video_clip, video_type="reel"):
    try:
        temp_dir = tempfile.mkdtemp()
        video = VideoFileClip(video_clip)
        audio = AudioFileClip(voiceover)
        duration = audio.duration
        
        # Resize based on video type
        if video_type == "reel":
            duration = min(duration, 60)
            video = video.resize((1080, 1920))  # 9:16
        else:
            duration = min(duration, 300)
            video = video.resize((1920, 1080))  # 16:9
        
        video = video.subclip(0, duration).set_audio(audio.subclip(0, duration))
        
        # Subtitles
        def make_textclip(txt):
            return TextClip(txt, fontsize=40, color="white", font="Arial", 
                           bg_color="black", size=(video.w, 100))
        
        subtitle_clip = SubtitlesClip(subtitles, make_textclip)
        subtitle_clip = subtitle_clip.set_pos(('center', 'bottom')).set_duration(duration)
        final = CompositeVideoClip([video, subtitle_clip])
        
        output_path = os.path.join(temp_dir, f"final_video_{uuid.uuid4()}.mp4")
        final.write_videofile(output_path, codec="libx264", audio_codec="aac")
        return output_path
    except Exception as e:
        st.error(f"Video creation failed: {str(e)}")
        return None
    finally:
        shutil.rmtree(temp_dir, ignore_errors=True)

# ===== STREAMLIT UI =====
def main():
    st.title("🎥 Free AI Video Generator (No APIs Needed)")
    st.markdown("Converts text to video using **free/local tools only** (gTTS + Whisper)")

    # User Inputs
    user_text = st.text_area("Enter your script:", "Hello, this is a free AI-generated video!")
    
    col1, col2 = st.columns(2)
    with col1:
        video_type = st.selectbox("Video Type", ["Reel (9:16)", "Long Video (16:9)"])
    with col2:
        subtitle_color = st.selectbox("Subtitle Color", ["white", "yellow", "blue"])
    
    if st.button("Generate Video"):
        with st.spinner("Creating your video..."):
            # Step 1: Voiceover (gTTS)
            voiceover = text_to_speech(user_text)
            if not voiceover:
                st.stop()
            
            # Step 2: Subtitles (Whisper)
            subtitles = generate_subtitles(voiceover)
            if not subtitles:
                st.stop()
            
            # Step 3: Local Video
            video_clip = get_local_video(video_type.split()[0].lower())
            if not video_clip:
                st.stop()
            
            # Step 4: Create Final Video
            final_video = create_video(
                voiceover, 
                subtitles, 
                video_clip, 
                video_type=video_type.split()[0].lower()
            )
            
            if final_video:
                st.success("Done!")
                st.video(final_video)
                with open(final_video, "rb") as f:
                    st.download_button("Download Video", f, file_name="ai_video.mp4")
            else:
                st.error("Failed to generate video. Please try again.")

if __name__ == "__main__":
    main()
