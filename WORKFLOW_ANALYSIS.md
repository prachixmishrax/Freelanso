# Post Upload Workflow Analysis

## 🔄 Complete Flow Overview

```
User Upload → File Validation → Cloudinary CDN → Express API → Firestore DB 
    ↓                                                            ↓
Local Preview                                           Fetch & Merge Data
                                                               ↓
                                                       ProjectCard Render
                                                               ↓
                                                       Display in /explore
```

---

## 📋 STAGE 1: File Selection & Validation (Frontend)

**File: [src/pages/PostWork.jsx](src/pages/PostWork.jsx)**

### State Management
```javascript
const [form, setForm] = useState({
  title: '', category: '', level: '', description: '',
  price: '', delivery: 'standard', skills: [], 
  image: null,          // 📝 Base64 preview (for UI only)
  mediaType: null,      // 📝 Stores 'image' or 'video'
})

const [rawFile, setRawFile] = useState(null) // 📝 Actual File object (critical!)
```

### loadMedia Function
```javascript
const loadMedia = (file) => {
  // 1️⃣ Validate file type
  const isImage = file.type.startsWith('image/')
  const isVideo = file.type.startsWith('video/')
  
  if (!isImage && !isVideo) {
    alert('Please select an image or video')
    return
  }
  
  // 2️⃣ Store File object reference (CRITICAL)
  setRawFile(file)  // ✅ Keep original file reference
  
  // 3️⃣ Create preview for UI (NOT for database)
  const reader = new FileReader()
  reader.onload = (e) => {
    set('image', e.target.result)  // Base64 for preview only
    set('mediaType', isImage ? 'image' : 'video')
  }
  reader.readAsDataURL(file)
}
```

### Key Points
- ✅ `rawFile` = Original File object (used for Cloudinary)
- ✅ `form.image` = Base64 string (used for UI preview only, NOT database)
- ✅ File validation happens before anything else
- ⚠️ Base64 can be 1MB+, would exceed Firestore's field limit

---

## 📤 STAGE 2: Upload to Cloudinary (Frontend)

**File: [src/lib/cloudinary.js](src/lib/cloudinary.js)**

### uploadMedia Function
```javascript
export const uploadMedia = async (file) => {
  // Get Cloudinary credentials from .env
  const cloudName = import.meta.env.VITE_CLOUDINARY_CLOUD_NAME
  const uploadPreset = import.meta.env.VITE_CLOUDINARY_UPLOAD_PRESET
  
  // Determine resource type
  const isVideo = file.type.startsWith('video/')
  const resourceType = isVideo ? 'video' : 'image'
  
  // Build FormData (NOT JSON - multipart/form-data)
  const formData = new FormData()
  formData.append('file', file)
  formData.append('upload_preset', uploadPreset)
  formData.append('folder', 'freelanso/uploads')
  formData.append('resource_type', resourceType)
  
  // POST directly to Cloudinary (bypasses backend)
  const endpoint = `https://api.cloudinary.com/v1_1/${cloudName}/${resourceType}/upload`
  const response = await fetch(endpoint, { method: 'POST', body: formData })
  
  // Return structured response
  return {
    url: data.secure_url,           // ✅ Permanent Cloudinary URL
    mediaType: isVideo ? 'video' : 'image',
    resourceType: data.resource_type,
    width: data.width,
    height: data.height,
    duration: data.duration  // For videos
  }
}
```

### Diagram
```
Browser
   ↓ FormData (multipart)
   ↓ (NOT JSON)
Cloudinary API
   ↓ Returns secure_url
Client
   ✅ https://res.cloudinary.com/freelanso/image/upload/...
```

### Error Handling
```javascript
// If Cloudinary fails → Fallback to picsum.photos
if (!response.ok) {
  console.error('Cloudinary upload failed')
  return {
    url: `https://picsum.photos/seed/${Date.now()}/800/600`,
    mediaType: isImage ? 'image' : 'video',
    fallback: true
  }
}
```

### Why Direct Upload?
- ✅ Bypasses Vercel's 4.5MB limit
- ✅ Bypasses Firebase CORS issues
- ✅ Offloads storage to CDN
- ✅ User sees secure_url immediately

---

## 💾 STAGE 3: Build & Send Payload (Frontend)

**File: [src/pages/PostWork.jsx - handleSubmit](src/pages/PostWork.jsx)**

### Payload Construction (✅ CRITICAL FIX)
```javascript
const payload = {
  title: form.title,
  category: form.category || '🎨 Design',
  level: form.level,
  description: form.description,
  price: `₹${Number(form.price).toLocaleString()}`,
  delivery: form.delivery,
  skills: form.skills,
  imageUrl: finalMediaUrl,    // ✅ Only the Cloudinary URL
  mediaType: finalMediaType,  // ✅ 'image' or 'video'
  author: user?.name || 'Anonymous User',
  authorUid: user?.uid,
  likes: 0,
  
  // ❌ NEVER include form.image (base64 string)
  // It would be 1MB+ and exceed Firestore's field limit
}
```

### Flow
```
uploadMedia(rawFile)
   ↓
finalMediaUrl = "https://res.cloudinary.com/..." ✅
   ↓
Build payload (imageUrl only, not form.image)
   ↓
saveProjectDB(payload)
```

### Why Not Include form.image?
- ❌ Base64 strings are 2-3x larger than original
- ❌ Firestore has 1MB field size limit
- ❌ No need - Cloudinary URL is permanent and efficient

---

## 🔐 STAGE 4: Backend Authentication & Storage

**File: [backend/server.js](backend/server.js)**

### Express Endpoint
```javascript
app.post('/api/projects', authenticate, async (req, res) => {
  try {
    // 1️⃣ Middleware verifies Firebase token
    // req.user = decodedToken with uid
    
    // 2️⃣ Build document
    const project = {
      ...req.body,              // Everything from payload
      authorUid: req.user.uid,  // Set by middleware
      createdAt: admin.firestore.FieldValue.serverTimestamp()
    }
    
    // 3️⃣ Add to Firestore
    const docRef = await db.collection('projects').add(project)
    
    // 4️⃣ Return response
    res.status(201).json({ id: docRef.id, ...project })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})
```

### Database Schema (Firestore)
```javascript
projects/{docId}
├── title: string
├── category: string
├── level: string
├── description: string
├── price: string (formatted with ₹)
├── imageUrl: string (Cloudinary URL) ✅
├── mediaType: string ('image' or 'video')
├── author: string (user name)
├── authorUid: string (user id)
├── likes: number
├── delivery: string
├── skills: array
├── createdAt: timestamp
└── updatedAt: timestamp (on updates)
```

### File Size Comparison
```
form.image (base64):     ~1.2 MB  ❌ Exceeds limit
imageUrl (Cloudinary):   ~50 bytes ✅ Efficient
```

---

## 🔄 STAGE 5: Fetch & Display (Frontend)

**Files: [src/lib/data.js](src/lib/data.js) & [src/pages/Explore.jsx](src/pages/Explore.jsx)**

### Fetch Flow
```javascript
// GET /api/projects
const fetchProjects = async () => {
  const response = await fetch(`${API_BASE}/projects`)
  const dbProjects = await response.json()
  
  // Merge with static projects
  return [...dbProjects, ...staticProjects]
}

// Explore.jsx loads on mount
useEffect(() => {
  const data = await fetchProjects()
  setAllProjects(data)  // Includes Firestore + static
}, [])
```

### Data Returned
```javascript
// Each project from database:
{
  id: "qbldHNQAxHwAonnyyoeH",
  title: "logo",
  category: "Logo Design",
  price: "₹500",
  imageUrl: "https://res.cloudinary.com/...",  // ✅ Actual Cloudinary URL
  mediaType: "image",
  author: "Username",
  authorUid: "user-uid",
  likes: 0,
  createdAt: timestamp,
  ...otherFields
}
```

### ProjectCard Rendering

**File: [src/components/ProjectCard.jsx](src/components/ProjectCard.jsx)**

```javascript
const displayImageUrl = project.imageUrl || project.img || `https://picsum.photos/...`
const isVideo = project.mediaType === 'video'

return (
  <div className="project-card">
    {isVideo ? (
      <video src={displayImageUrl} />  // ✅ Cloudinary video URL
    ) : (
      <img src={displayImageUrl} />    // ✅ Cloudinary image URL
    )}
  </div>
)
```

### URL Resolution Priority
```
1. project.imageUrl    (Cloudinary URL from database)  ✅ First choice
   ↓ (if missing)
2. project.img         (Static project fallback)
   ↓ (if missing)
3. Fallback picsum.photos
```

---

## 🐛 Common Issues & Fixes

### Issue #1: Base64 Exceeds Firestore Limit
```
Error: The value of property "image" is longer than 1048487 bytes
```
**Root Cause:** Spreading entire form object (`...form`) in payload  
**Fix:** Only include individual fields, exclude `image` field
```javascript
// ❌ Wrong
const payload = { ...form, imageUrl, ...}

// ✅ Right
const payload = { 
  title: form.title,
  imageUrl: finalMediaUrl,  // Only the URL
  // No form.image
}
```

### Issue #2: Wrong File Uploaded
```
Upload file A → See file B in feed
```
**Root Cause:** File input not reset after publish  
**Fix:** Reset file input and rawFile state
```javascript
// After successful publish
setForm({ ...resetFields })
setRawFile(null)
const fileInput = document.getElementById('file-input')
if (fileInput) fileInput.value = ''
```

### Issue #3: Cloudinary 401 Unauthorized
```
Error: Unknown API key
```
**Root Cause:** Missing or invalid env variables  
**Fix:** Verify .env has:
```env
VITE_CLOUDINARY_CLOUD_NAME=your_cloud_name
VITE_CLOUDINARY_UPLOAD_PRESET=your_preset
```

### Issue #4: Video Doesn't Play in Feed
**Root Cause:** mediaType not stored in database  
**Fix:** Ensure mediaType is included in payload when posting/editing

---

## 📊 Data Flow Summary

```
FRONTEND (Browser)
├── User selects file
├── loadMedia() validates & stores in state
│   ├── rawFile = File object
│   └── form.image = Base64 preview
├── User fills form & publishes
└── handleSubmit()
    ├── uploadMedia(rawFile) → Cloudinary
    ├── Build payload (imageUrl only)
    └── saveProjectDB(payload)
        │
        └─→ EXPRESS API (Backend)
            ├── Authenticate (verify token)
            └── db.collection('projects').add()
                │
                └─→ FIRESTORE (Database)
                    └── Document stored ✅
                        │
                        └─→ API responds with ID
                            │
                            └─→ FRONTEND receives ID
                                ├── Form reset
                                ├── Navigate to /explore
                                │
                                └─→ EXPLORE PAGE
                                    ├── fetchProjects()
                                    ├── GET /api/projects
                                    │
                                    └─→ API queries Firestore
                                        ├── Sort by createdAt DESC
                                        ├── Map to array
                                        └── Return with IDs
                                            │
                                            └─→ FRONTEND renders
                                                ├── setAllProjects(data)
                                                ├── Loop: ProjectCard
                                                │
                                                └─→ DISPLAY
                                                    └── Your post ✅
```

---

## 🔍 Debugging Checklist

- [ ] File selected shows in console with name & size
- [ ] loadMedia logs: "📁 File selected: name (size MB, type)"
- [ ] Cloudinary upload logs: "📤 Uploading to Cloudinary..."
- [ ] Success log: "✅ Successfully received Cloudinary URL: https://res..."
- [ ] Payload logged: imageUrl is Cloudinary URL, not base64
- [ ] Database saved: Check Firestore projectsindex collection
- [ ] Post appears in feed: Check /explore page
- [ ] Correct image shown: Verify imageUrl matches uploaded file

---

## ✅ Expected Console Logs (Success Flow)

```
📁 File selected: photo.jpg (1.01MB, type: image/jpeg)
👁️ Preview loaded for: photo.jpg
🔝 About to upload file: photo.jpg (1.01MB, type: image/jpeg)
📤 Uploading media to Cloudinary...
📤 Uploading image to Cloudinary... photo.jpg
✅ Successfully received Cloudinary URL: https://res.cloudinary.com/...
💾 Saving project to database: { title: '...', imageUrl: 'https://...', mediaType: 'image' }
API: Saving project to Express... logo
API: Project saved with ID: qbldHNQAxHwAonnyyoeH
✅ Project published successfully!
```

---

## 🎯 Summary

| Stage | File | Function | Output |
|-------|------|----------|--------|
| 1. Select | PostWork.jsx | loadMedia() | rawFile state |
| 2. Preview | PostWork.jsx | FileReader | form.image (base64) |
| 3. Upload | cloudinary.js | uploadMedia() | Cloudinary URL |
| 4. Build | PostWork.jsx | handleSubmit() | Payload (imageUrl only) |
| 5. Store | server.js | POST /projects | Firestore document |
| 6. Fetch | data.js | fetchProjects() | Project array |
| 7. Display | ProjectCard.jsx | Component render | Post in feed ✅ |

# Status notice

This document describes the former Express API workflow. The backend has been removed; upload-to-Cloudinary remains a frontend flow, but API persistence and backend-dependent operations documented below are not available in the current frontend-only project.
