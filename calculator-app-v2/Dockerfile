FROM python:3.11-slim

WORKDIR /app

# העתקה והתקנה של הדרישות
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# העתקת שאר קבצי האפליקציה
COPY . .

# חשיפת הפורט של האפליקציה כפי שמופיע ב-README
EXPOSE 5000

# הרצת האפליקציה (לפי קובץ ה-API או ה-App הראשי)
CMD ["python", "api.py"]
