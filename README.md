rom flask import Flask, render_template_string, request, redirect, Response
import cv2, threading, time, os, datetime
import speech_recognition as sr
import pyttsx3
from PyPDF2 import PdfReader
 
app = Flask(__name__)
 
# ================= GLOBAL STATE =================
screening_active = False
latest_transcript = "Waiting for response..."
answers = []
current_question = 0
 
questions = [
   "Tell me about yourself",
   "Explain a project you worked on",
   "What are your technical strengths?",
   "Why should we hire you?"
]
 
user_profile = {}
video_writer = None
video_filename = None
 
# ================= CAMERA GLOBAL =================
camera = cv2.VideoCapture(0)
 
# ================= CREATE RECORDINGS DIR =================
if not os.path.exists("recordings"):
  os.mkdir("recordings")
 
# ================= TTS =================
engine = pyttsx3.init()
def speak(text):
   engine.say(text)
  engine.runAndWait()
 
# ================= STT =================
def listen_voice():
   global latest_transcript
   r = sr.Recognizer()
   mic = sr.Microphone()
 
   while True:
       if not screening_active:
          time.sleep(0.2)
           continue
 
       try:
           with mic as source:
              r.adjust_for_ambient_noise(source, 0.3)
               audio = r.listen(source, timeout=5, phrase_time_limit=7)
              latest_transcript = r.recognize_google(audio)
       except:
           pass
 
# ================= CAMERA =================
face_cascade = cv2.CascadeClassifier(cv2.data.haarcascades+"haarcascade_frontalface_default.xml")
smile_cascade = cv2.CascadeClassifier(cv2.data.haarcascades+"haarcascade_smile.xml")
 
DARK_BLUE=(160,60,60)
DARK_GREEN=(0,120,0)
 
# analysis metrics
confidence_score=0
smile_score=0
frames_count=0
 
def gen_frames():
   global video_writer, confidence_score, smile_score, frames_count
 
   while True:
      success,frame=camera.read()
       if not success:
           continue
 
      frames_count+=1
 
      gray=cv2.cvtColor(frame,cv2.COLOR_BGR2GRAY)
      faces=face_cascade.detectMultiScale(gray,1.3,5)
 
      direction_text="No Face"
      smile_text="Not Smiling 😐"
 
       if len(faces)>0:
          confidence_score+=1
 
          x,y,w,h=faces[0]
          frame_h,frame_w=frame.shape[:2]
           cx=x+w//2
           cy=y+h//2
          nx=cx/frame_w
          ny=cy/frame_h
 
           if nx<0.4: lr="Left"
           elif nx>0.6: lr="Right"
           else: lr="Center"
 
           if ny<0.4: ud="Up"
           elif ny>0.6: ud="Down"
           else: ud="Center"
 
          direction_text=f"Looking {lr}-{ud}"
 
          roi_gray=gray[y:y+h,x:x+w]
          mouth_roi=roi_gray[int(h*0.6):h,:]
 
          smiles=smile_cascade.detectMultiScale(mouth_roi,1.5,10,minSize=(25,15))
           if len(smiles)>0:
              smile_score+=1
              smile_text="Smiling 😊"
 
          cv2.rectangle(frame,(x,y),(x+w,y+h),(0,200,0),2)
 
      cv2.putText(frame,direction_text,(30,40),cv2.FONT_HERSHEY_SIMPLEX,0.9,DARK_BLUE,2)
      cv2.putText(frame,smile_text,(30,80),cv2.FONT_HERSHEY_SIMPLEX,0.9,DARK_GREEN,2)
 
       if screening_active and video_writer:
          video_writer.write(frame)
 
      ret,buffer=cv2.imencode('.jpg',frame)
      frame=buffer.tobytes()
 
      yield(b'--frame\r\n'
            b'Content-Type: image/jpeg\r\n\r\n'+frame+b'\r\n')
 
@app.route("/video_feed")
def video_feed():
   return Response(gen_frames(),mimetype="multipart/x-mixed-replace; boundary=frame")
 
# ================= INTERVIEW FLOW =================
def interview_flow():
   global current_question, screening_active, latest_transcript
 
   while True:
       if not screening_active:
          time.sleep(0.2)
           continue
 
      speak(questions[current_question])
      latest_transcript="Listening..."
       time.sleep(7)
 
# ================= NEXT QUESTION =================
@app.route("/next")
def next_question():
   global current_question, latest_transcript
 
   answers.append({
      "question":questions[current_question],
      "answer":latest_transcript
   })
 
   current_question += 1
   if current_question >= len(questions):
      current_question = 0
 
  threading.Thread(target=speak, args=(questions[current_question],), daemon=True).start()
   return redirect("/screening")
 
# ================= UI BASE =================
BASE_HTML="""
<!DOCTYPE html>
<html>
<head>
<title>AI Mock Interview</title>
<style>
body{font-family:Segoe UI;background:#f1f5f9;margin:0}
nav{background:#1e3a8a;padding:16px;text-align:center}
nav a{color:white;margin:0 20px;text-decoration:none;font-weight:600}
.container{width:1100px;margin:40px auto}
.card{background:white;padding:25px;border-radius:16px;
box-shadow:0 15px 40px rgba(0,0,0,.12);margin-bottom:25px}
button{padding:12px 30px;border:none;border-radius:10px;
background:#2563eb;color:white;font-size:16px;cursor:pointer}
.green{background:#16a34a}
.red{background:#dc2626}
.orange{background:#f59e0b}
.hero{text-align:center}
.score{font-size:22px;font-weight:bold}
</style>
</head>
<body>
 
<nav>
<a href="/">Home</a>
<a href="/profile">Profile</a>
<a href="/dashboard">Dashboard</a>
<a href="/screening">Screening</a>
<a href="/feedback">Feedback</a>
</nav>
 
<div class="container">
{{content|safe}}
</div>
</body>
</html>
"""
 
# ================= HOME =================
@app.route("/")
def home():
   return render_template_string(BASE_HTML,content="""
   <div class="hero">
       <h1>AI Mock Interview</h1>
       <a href="/profile"><button>Start Interview</button></a>
   </div>
  """)
 
# ================= PROFILE =================
def extract_text_from_pdf(file):
  reader=PdfReader(file)
   return "".join(page.extract_text() for page in reader.pages if page.extract_text())
 
@app.route("/profile",methods=["GET","POST"])
def profile():
   global user_profile
   if request.method=="POST":
      user_profile["name"]=request.form["name"]
      user_profile["edu"]=request.form["edu"]
      user_profile["cgpa"]=request.form["cgpa"]
      user_profile["resume"]=extract_text_from_pdf(request.files["resume"])
      user_profile["jd"]=extract_text_from_pdf(request.files["jd"])
       return redirect("/dashboard")
 
   return render_template_string(BASE_HTML,content="""
  <h2>Candidate Profile</h2>
   <form method="post" enctype="multipart/form-data">
       <input name="name" placeholder="Full Name" required>
       <input name="edu" placeholder="Education" required>
       <input name="cgpa" placeholder="CGPA" required>
       <input type="file" name="resume" accept=".pdf" required>
       <input type="file" name="jd" accept=".pdf" required>
      <button>Save</button>
   </form>
  """)
 
# ================= DASHBOARD =================
@app.route("/dashboard")
def dashboard():
   return render_template_string(BASE_HTML,content="""
   <h2>ATS Evaluation</h2>
   <div class="card"><b>Match:</b> 78%</div>
   <a href="/start"><button class="green">Start Screening</button></a>
  """)
 
# ================= START =================
@app.route("/start")
def start():
   global screening_active,current_question,answers,video_writer,video_filename
 
  screening_active=True
   current_question=0
   answers=[]
 
  ts=datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
  video_filename=f"recordings/interview_{ts}.mp4"
  fourcc=cv2.VideoWriter_fourcc(*"mp4v")
  video_writer=cv2.VideoWriter(video_filename,fourcc,20.0,(640,480))
 
   return redirect("/screening")
 
# ================= STOP =================
@app.route("/stop")
def stop():
   global screening_active,video_writer
  screening_active=False
   if video_writer:
      video_writer.release()
      video_writer=None
   return redirect("/feedback")
 
# ================= SCREENING =================
@app.route("/screening")
def screening():
  q=questions[current_question]
 
   return render_template_string(BASE_HTML,content=f"""
   <meta http-equiv="refresh" content="2">
   <h2>Live Screening</h2>
 
   <div style="display:flex;gap:20px">
 
       <div style="width:50%">
           <div class="card">
              <h3>Question</h3>
              <b>{q}</b>
          </div>
 
           <a href="/start"><button class="green">Start</button></a>
           <a href="/next"><button class="orange">Next</button></a>
           <a href="/stop"><button class="red">Stop</button></a>
       </div>
 
       <div style="width:50%">
           <div class="card">
              <h3>Camera</h3>
              <img src="/video_feed" width="100%">
          </div>
           <div class="card">
              <h3>Speech</h3>
              {latest_transcript}
          </div>
       </div>
 
   </div>
  """)
 
# ================= FEEDBACK =================
@app.route("/feedback")
def feedback():
 
  qa="".join(f"<li><b>{a['question']}</b><br>{a['answer']}</li>" for a in answers)
 
   # -------- ANALYSIS SCORES --------
   confidence = int((confidence_score/frames_count)*100) if frames_count else 0
   smile = int((smile_score/frames_count)*100) if frames_count else 0
   communication = min(100,len(" ".join(a["answer"] for a in answers))*2)
   grooming = (confidence + smile)//2
   personality = (confidence + communication + grooming)//3
 
   # overall verdict
   if personality>75:
      verdict="Excellent Candidate ⭐"
   elif personality>50:
      verdict="Good Candidate 👍"
   else:
      verdict="Needs Improvement ⚠"
 
   return render_template_string(BASE_HTML,content=f"""
  <h2>Interview Analysis Report</h2>
 
   <div class="card">
      <h3>Answers</h3>
      <ul>{qa}</ul>
   </div>
 
   <div class="card">
      <h3>Performance Scores</h3>
       Confidence : <div class="score">{confidence}%</div>
       Communication : <div class="score">{communication}%</div>
       Grooming : <div class="score">{grooming}%</div>
       Smile Confidence : <div class="score">{smile}%</div>
       Personality : <div class="score">{personality}%</div>
   </div>
 
   <div class="card">
      <h3>Final AI Verdict</h3>
      <h2>{verdict}</h2>
   </div>
 
   <div class="card">
      <h3>Recorded Video</h3>
      {video_filename}
   </div>
  """)
 
# ================= THREAD START =================
threading.Thread(target=listen_voice,daemon=True).start()
threading.Thread(target=interview_flow,daemon=True).start()
 
# ================= RUN =================
if __name__=="__main__":
  app.run(debug=True)
  powershell: pip install flask opencv-python pyttsx3 SpeechRecognition pdfplumber pyaudio
