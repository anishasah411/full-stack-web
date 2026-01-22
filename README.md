#APP.JS
import { useState, useEffect } from "react";
import Login from "./auth";
import Upload from "./upload";
import BeforeAfter from "./components";
import { getUploadsByUser } from "./firebase";

function App() {
  const [user, setUser] = useState(null);
  const [original, setOriginal] = useState(null);
  const [processed, setProcessed] = useState(null);
  const [gallery, setGallery] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");

  useEffect(() => {
    if (user) {
      getUploadsByUser(user.uid)
        .then((data) => setGallery(data))
        .catch((err) => setError(err.message));
    }
  }, [user]);

  return (
    <div style={{ padding: 20 }}>
      <Login setUser={setUser} user={user} />

      {user && (
        <>
          <Upload
            setOriginal={setOriginal}
            setProcessed={setProcessed}
            user={user}
          />

          {loading && <p>Processing image...</p>}
          {error && <p style={{ color: "red" }}>{error}</p>}

          {original && (
            <>
              <BeforeAfter original={original} processed={processed} />
            </>
          )}

          <h3>Your Uploads:</h3>
          <div style={{ display: "flex", gap: 20, flexWrap: "wrap" }}>
            {gallery
              .sort((a, b) => b.createdAt?.seconds - a.createdAt?.seconds)
              .map((item) => (
                <div key={item.id}>
                  <p>{item.filename}</p>
                  <img src={item.originalUrl} width="100" alt="original" />
                  {item.processedUrl && (
                    <img src={item.processedUrl} width="100" alt="processed" />
                  )}
                </div>
              ))}
          </div>
        </>
      )}
    </div>
  );
}

export default App;
#AUTH.JS
import { useState, useEffect } from "react";
import { auth, signIn, signUp, signOutUser } from "./firebase";

export default function Login({ setUser, user }) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  useEffect(() => {
    const unsubscribe = auth.onAuthStateChanged(setUser);
    return () => unsubscribe();
  }, [setUser]);

  const handleLogin = async () => {
    try {
      const res = await signIn(email, password);
      setUser(res.user);
    } catch (err) {
      alert(err.message);
    }
  };

  const handleSignUp = async () => {
    try {
      const res = await signUp(email, password);
      setUser(res.user);
    } catch (err) {
      alert(err.message);
    }
  };

  if (user) {
    return (
      <div>
        <p>Signed in as {user.email}</p>
        <button onClick={signOutUser}>Sign Out</button>
      </div>
    );
  }

  return (
    <div>
      <h3>Login / Sign Up</h3>
      <input
        placeholder="Email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button onClick={handleLogin}>Login</button>
      <button onClick={handleSignUp}>Sign Up</button>
    </div>
  );
}
UPLOAD.JS:
import { uploadFile } from "./firebase";
import * as pdfjsLib from "pdfjs-dist/build/pdf";

/* ---------------- PDF.js worker (CDN) ---------------- */
pdfjsLib.GlobalWorkerOptions.workerSrc =
  `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.js`;

/* ---------------- Wait for OpenCV.js to load safely ---------------- */
function waitForOpenCV() {
  return new Promise((resolve) => {
    const check = () => {
      if (window.cv && window.cv.imread) resolve();
      else setTimeout(check, 100);
    };
    check();
  });
}

/* ======================= Upload Component ======================= */
export default function Upload({ setOriginal, setProcessed, user }) {
  const handleFile = async (e) => {
    const file = e.target.files[0];
    if (!file || !user) return;

    /* ======================= PDF Upload ======================= */
    if (file.type === "application/pdf") {
      const buffer = await file.arrayBuffer();
      const pdf = await pdfjsLib.getDocument({ data: buffer }).promise;
      const page = await pdf.getPage(1);

      const viewport = page.getViewport({ scale: 2 });
      const canvas = document.createElement("canvas");
      const ctx = canvas.getContext("2d");

      canvas.width = viewport.width;
      canvas.height = viewport.height;

      await page.render({ canvasContext: ctx, viewport }).promise;

      const imgUrl = canvas.toDataURL("image/png");
      setOriginal(imgUrl);

      await waitForOpenCV();
      const cropped = await autoCrop(imgUrl);
      setProcessed(cropped);

      /* Upload rasterized PDF page as image */
      const blob = await (await fetch(imgUrl)).blob();
      const imageFile = new File([blob], "page1.png", { type: "image/png" });

      await uploadFile(imageFile, user.uid, cropped);
    }

    /* ======================= Image Upload ======================= */
    else {
      const url = URL.createObjectURL(file);
      setOriginal(url);

      await waitForOpenCV();
      const cropped = await autoCrop(url);
      setProcessed(cropped);

      await uploadFile(file, user.uid, cropped);
    }
  };

  return (
    <input
      type="file"
      accept="image/*,application/pdf"
      onChange={handleFile}
    />
  );
}

/* ======================= Auto Crop (OpenCV.js) ======================= */
export async function autoCrop(imgUrl) {
  return new Promise((resolve) => {
    const img = new Image();
    img.src = imgUrl;

    img.onload = () => {
      try {
        const mat = cv.imread(img);

        let gray = new cv.Mat();
        cv.cvtColor(mat, gray, cv.COLOR_RGBA2GRAY);

        let blur = new cv.Mat();
        cv.GaussianBlur(gray, blur, new cv.Size(5, 5), 0);

        let edges = new cv.Mat();
        cv.Canny(blur, edges, 75, 200);

        let contours = new cv.MatVector();
        let hierarchy = new cv.Mat();
        cv.findContours(edges, contours, hierarchy, cv.RETR_LIST, cv.CHAIN_APPROX_SIMPLE);

        let maxArea = 0;
        let pageContour = null;

        for (let i = 0; i < contours.size(); i++) {
          const cnt = contours.get(i);
          const peri = cv.arcLength(cnt, true);
          const approx = new cv.Mat();
          cv.approxPolyDP(cnt, approx, 0.02 * peri, true);

          if (approx.rows === 4) {
            const area = cv.contourArea(approx);
            if (area > maxArea) {
              maxArea = area;
              pageContour = approx;
            }
          }
        }

        /* ---- Fail-safe fallback ---- */
        if (!pageContour) {
          resolve(imgUrl);
          mat.delete(); gray.delete(); blur.delete(); edges.delete();
          contours.delete(); hierarchy.delete();
          return;
        }

        /* ---- Order points ---- */
        const pts = [];
        for (let i = 0; i < 4; i++) {
          pts.push({
            x: pageContour.intPtr(i, 0)[0],
            y: pageContour.intPtr(i, 0)[1],
          });
        }

        pts.sort((a, b) => a.y - b.y);
        const top = pts.slice(0, 2).sort((a, b) => a.x - b.x);
        const bottom = pts.slice(2, 4).sort((a, b) => b.x - a.x);
        const ordered = [top[0], top[1], bottom[0], bottom[1]];

        const widthA = Math.hypot(
          ordered[2].x - ordered[3].x,
          ordered[2].y - ordered[3].y
        );
        const widthB = Math.hypot(
          ordered[1].x - ordered[0].x,
          ordered[1].y - ordered[0].y
        );
        const maxWidth = Math.max(widthA, widthB);

        const heightA = Math.hypot(
          ordered[1].x - ordered[2].x,
          ordered[1].y - ordered[2].y
        );
        const heightB = Math.hypot(
          ordered[0].x - ordered[3].x,
          ordered[0].y - ordered[3].y
        );
        const maxHeight = Math.max(heightA, heightB);

        const srcTri = cv.matFromArray(4, 1, cv.CV_32FC2, [
          ordered[0].x, ordered[0].y,
          ordered[1].x, ordered[1].y,
          ordered[2].x, ordered[2].y,
          ordered[3].x, ordered[3].y,
        ]);

        const dstTri = cv.matFromArray(4, 1, cv.CV_32FC2, [
          0, 0,
          maxWidth - 1, 0,
          maxWidth - 1, maxHeight - 1,
          0, maxHeight - 1,
        ]);

        const M = cv.getPerspectiveTransform(srcTri, dstTri);
        const dst = new cv.Mat();
        cv.warpPerspective(mat, dst, M, new cv.Size(maxWidth, maxHeight));

        const canvas = document.createElement("canvas");
        cv.imshow(canvas, dst);

        const croppedUrl = canvas.toDataURL("image/png");

        mat.delete(); gray.delete(); blur.delete(); edges.delete();
        contours.delete(); hierarchy.delete(); pageContour.delete();
        srcTri.delete(); dstTri.delete(); M.delete(); dst.delete();

        resolve(croppedUrl);
      } catch (err) {
        console.error(err);
        resolve(imgUrl);
      }
    };

    img.onerror = () => resolve(imgUrl);
  });
}
#COMPONENTS.JS
export default function BeforeAfter({ original, processed }) {
  return (
    <div style={{ display: "flex", gap: 20, marginTop: 20 }}>
      {original && (
        <div>
          <p>Before</p>
          <img src={original} width="200" alt="Before" />
        </div>
      )}
      {processed && (
        <div>
          <p>After</p>
          <img src={processed} width="200" alt="After" />
        </div>
      )}
    </div>
  );
}
#FIREBASE.JS
import { initializeApp } from "firebase/app";
import {
  getAuth,
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  signOut,
} from "firebase/auth";
import {
  getFirestore,
  collection,
  addDoc,
  getDocs
} from "firebase/firestore";
import { getStorage, ref, uploadBytes, getDownloadURL } from "firebase/storage";

// Firebase config
const firebaseConfig = {
  apiKey: "AIzaSyCIExLCQC6HMTwDdP0ee5WspnscKDhitaI",
  authDomain: "document-scanner-aa83a.firebaseapp.com",
  projectId: "document-scanner-aa83a",
  storageBucket: "document-scanner-aa83a.firebasestorage.app",
  messagingSenderId: "708575198068",
  appId: "1:708575198068:web:e18ab25e402f8f63ee0a21",
};

const app = initializeApp(firebaseConfig);

export const auth = getAuth(app);
export const db = getFirestore(app);
export const storage = getStorage(app);

// Auth helpers
export const signIn = (email, password) =>
  signInWithEmailAndPassword(auth, email, password);

export const signUp = (email, password) =>
  createUserWithEmailAndPassword(auth, email, password);

export const signOutUser = () => signOut(auth);

// ✅ Upload file + save metadata (STEP 2 COMPLETE)
export async function uploadFile(file, userId) {
  const storageRef = ref(storage, `uploads/${userId}/${file.name}`);

  await uploadBytes(storageRef, file);

  const fileURL = await getDownloadURL(storageRef);

  await addDoc(
    collection(db, "users", userId, "documents"),
    {
      fileName: file.name,
      fileURL: fileURL,
      uploadedAt: new Date(),
    }
  );
}

// ✅ Get uploaded files for logged-in user
export async function getUploadsByUser(userId) {
  const snapshot = await getDocs(
    collection(db, "users", userId, "documents")
  );

  return snapshot.docs.map((doc) => ({
    id: doc.id,
    ...doc.data(),
  }));
}
#GRAYSCALE.JS
import cv2

# Read the input image
input_image = 'input.jpg'  # replace with your uploaded file
img = cv2.imread(input_image)

# Convert to grayscale
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# Save the output image
output_image = 'output.jpg'
cv2.imwrite(output_image, gray)

print(f"Grayscale image saved as {output_image}")
APP.PY:
from flask import Flask, request, jsonify
import cv2
import numpy as np
import requests
from pdf2image import convert_from_path
import os
from PIL import Image

app = Flask(__name__)

# --- Convert PDF to image ---
def pdf_to_image(pdf_path):
    pages = convert_from_path(pdf_path, dpi=200)
    return pages[0]  # first page only

# --- Scan / perspective correction ---
def scan_document(image_path):
    img = cv2.imread(image_path)
    orig = img.copy()
    
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    gray = cv2.GaussianBlur(gray, (5,5), 0)
    edges = cv2.Canny(gray, 75, 200)
    
    contours, _ = cv2.findContours(edges, cv2.RETR_LIST, cv2.CHAIN_APPROX_SIMPLE)
    contours = sorted(contours, key=cv2.contourArea, reverse=True)
    
    doc_cnt = None
    for c in contours:
        peri = cv2.arcLength(c, True)
        approx = cv2.approxPolyDP(c, 0.02 * peri, True)
        if len(approx) == 4:
            doc_cnt = approx
            break
    if doc_cnt is None:
        return image_path  # return original if not found
    
    pts = doc_cnt.reshape(4,2)
    rect = np.zeros((4,2), dtype="float32")
    s = pts.sum(axis=1)
    rect[0] = pts[np.argmin(s)]
    rect[2] = pts[np.argmax(s)]
    diff = np.diff(pts, axis=1)
    rect[1] = pts[np.argmin(diff)]
    rect[3] = pts[np.argmax(diff)]
    
    (tl, tr, br, bl) = rect
    widthA = np.linalg.norm(br-bl)
    widthB = np.linalg.norm(tr-tl)
    maxWidth = max(int(widthA), int(widthB))
    heightA = np.linalg.norm(tr-br)
    heightB = np.linalg.norm(tl-bl)
    maxHeight = max(int(heightA), int(heightB))
    
    dst = np.array([[0,0],[maxWidth-1,0],[maxWidth-1,maxHeight-1],[0,maxHeight-1]], dtype="float32")
    M = cv2.getPerspectiveTransform(rect, dst)
    scanned = cv2.warpPerspective(orig, M, (maxWidth, maxHeight))
    
    output_path = "scanned.jpg"
    cv2.imwrite(output_path, scanned)
    return output_path

# --- Flask route ---
@app.route('/scan', methods=['POST'])
def scan_file():
    file_url = request.json['file_url']
    r = requests.get(file_url)
    
    # save temp file
    temp_path = "temp_file"
    with open(temp_path, 'wb') as f:
        f.write(r.content)
    
    # if PDF, convert to image
    if file_url.lower().endswith(".pdf"):
        img = pdf_to_image(temp_path)
        temp_path = "temp_file.jpg"
        img.save(temp_path)
    
    scanned_path = scan_document(temp_path)
    
    # NOTE: For now, we just return the scanned image as a local path
    # Later we can upload to Firebase
    return jsonify({"scanned_path": scanned_path})

if __name__ == "__main__":
    app.run(debug=True)
    #INDEX.HTML
    <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Document Scanner</title>

  <!-- OpenCV.js -->
  <script async src="https://docs.opencv.org/4.x/opencv.js" type="text/javascript"></script>
</head>
<body>
  <div id="root"></div>
</body>
</html>


