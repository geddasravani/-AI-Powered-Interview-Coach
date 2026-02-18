# -AI-Powered-Interview-Coach
Current interview practice tools only record responses but do not analyze candidate behavior, confidence, or communication skills, and they lack real-time feedback.  Solution: An AI-based intelligent interviewer system that evaluates candidates automatically and provides instant performance analysis.
🎤 AI Mock Interview System
An intelligent AI-powered mock interview platform that simulates real interview environments by analyzing candidate responses, facial expressions, confidence level, and communication skills in real time.
📌 Project Overview
The AI Mock Interview System is a Flask-based web application that conducts automated interviews using:
Speech Recognition
Face Detection
Smile Detection
Video Recording
Performance Scoring
It evaluates candidates without a human interviewer and generates a final feedback report.
🚀 Features
✔ Automated interview questions
✔ Live camera monitoring
✔ Speech-to-Text response capture
✔ Facial expression tracking
✔ Confidence & personality scoring
✔ Resume + Job Description upload
✔ Recorded interview video
✔ AI performance verdict
🧠 How It Works
User enters profile details
Uploads Resume & Job Description PDFs
Starts interview session
System asks questions via voice
Camera tracks face & smile
Microphone records responses
AI calculates scores
Final report generated
🏗️ Tech Stack
Backend
Python
Flask
Threading
AI & Processing
OpenCV → Face detectio
Haar Cascades → Smile detection
SpeechRecognition → Speech-to-Text
pyttsx3 → Text-to-Speech
PyPDF2 → PDF parsing
Frontend
HTML
CSS
Live video streaming (MJPEG)
📂 Project Structure
project/
│
├── app.py
├── recordings/
│
├── README.md
⚙️ Installation
1️⃣ Clone Repository
git clone https://github.com/yourusername/ai-mock-interview.git
cd ai-mock-interview
2️⃣ Install Dependencies
pip install flask opencv-python speechrecognition pyttsx3 pypdf2 pyaudio
⚠ If PyAudio fails:
pip install pipwin
pipwin install pyaudio
3️⃣ Run Application
python app.py
Open browser:
http://127.0.0.1:5000
📊 Scoring Metrics
The system calculates:
Confidence Score → Face visibility
Smile Score → Positive expression
Communication Score → Response length
Grooming Score → Confidence + Smile
Personality Score → Overall average
📸 Output Example
Final Report Includes:
Questions & answers
Performance percentages
AI verdict
Recorded video path
🎯 Use Cases
Placement training
Interview preparation
HR screening
Communication skill improvement
Behavioral analysis research
🔮 Future Improvements
Emotion detection
Eye tracking
NLP answer evaluation
Cloud storage
Multi-language support
Dashboard analytics
⚠ Limitations
Sensitive to lighting conditions
Background noise affects speech accuracy
Requires camera & microphone
Depends on internet for speech recognition
🤝 Contribution
Pull requests are welcome.
For major changes, open an issue first to discuss improvements.
📜 License
This project is open-source and available under the MIT License.
🙌 Acknowledgement
Developed as an academic AI project to demonstrate real-time behavioral analysis in interviews.
