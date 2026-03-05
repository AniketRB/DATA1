import { useState, useEffect, useRef, useCallback } from 'react'
import { useNavigate } from 'react-router-dom'
import Webcam from 'react-webcam'
import { getUnknownFaces, clearUnknownFaces, getDepartments, registerFaculty } from '../services/api'
import PageHeader from '../components/ui/PageHeader'
import Btn from '../components/ui/Btn'

export default function UnknownFaces() {
  const [logs, setLogs] = useState([])
  const [loading, setLoading] = useState(true)
  const [clearing, setClearing] = useState(false)
  const [registerModal, setRegisterModal] = useState(null) // holds the log object
  const navigate = useNavigate()

  useEffect(() => { fetchLogs() }, [])

  const fetchLogs = async () => {
    try {
      const res = await getUnknownFaces()
      setLogs(res.data)
    } catch (e) { console.error(e) }
    finally { setLoading(false) }
  }

  const handleClearAll = async () => {
    if (!window.confirm('Clear all unknown face logs? This cannot be undone.')) return
    setClearing(true)
    try {
      await clearUnknownFaces()
      setLogs([])
    } catch (e) { console.error(e) }
    finally { setClearing(false) }
  }

  return (
    <div>
      <PageHeader
        title="Unknown Face Alerts"
        subtitle={`${logs.length} unrecognized face${logs.length !== 1 ? 's' : ''} detected`}
        actions={<>
          {logs.length > 0 && (
            <Btn variant="danger" onClick={handleClearAll} disabled={clearing}>
              {clearing ? 'Clearing...' : '🗑 Clear All'}
            </Btn>
          )}
        </>}
      />

      {/* Info banner */}
      <div style={{
        padding: '14px 18px', borderRadius: '10px', marginBottom: '24px',
        background: 'rgba(246,79,59,0.06)', border: '1px solid rgba(246,79,59,0.15)',
        display: 'flex', alignItems: 'center', gap: '12px'
      }}>
        <span style={{ fontSize: '18px' }}>ℹ</span>
        <div>
          <div style={{ fontSize: '13px', fontWeight: 500, color: '#F64F3B' }}>What are unknown alerts?</div>
          <div style={{ fontSize: '12px', color: '#5A6A85', marginTop: '2px' }}>
            Faces detected by the camera that don't match any registered faculty.
            Click "Register This Person" to register them directly from their snapshot.
          </div>
        </div>
      </div>

      {loading ? (
        <div style={{ textAlign: 'center', color: '#5A6A85', padding: '60px' }}>
          <div style={{ fontSize: '24px', marginBottom: '12px' }}>⏳</div>
          Loading alerts...
        </div>
      ) : logs.length === 0 ? (
        <div style={{
          textAlign: 'center', padding: '80px',
          background: '#0D1421', border: '1px solid rgba(255,255,255,0.06)',
          borderRadius: '14px'
        }}>
          <div style={{ fontSize: '48px', marginBottom: '16px' }}>✅</div>
          <div style={{ fontSize: '16px', fontWeight: 600, marginBottom: '8px' }}>All Clear</div>
          <div style={{ fontSize: '13px', color: '#5A6A85' }}>No unknown faces detected</div>
        </div>
      ) : (
        <div style={{ display: 'grid', gridTemplateColumns: 'repeat(auto-fill, minmax(220px, 1fr))', gap: '16px' }}>
          {logs.map((log, i) => (
            <div key={log.id || i} style={{
              background: '#0D1421', border: '1px solid rgba(246,79,59,0.2)',
              borderRadius: '14px', overflow: 'hidden'
            }}>
              <div style={{
                aspectRatio: '1', background: '#050810',
                display: 'flex', alignItems: 'center', justifyContent: 'center',
                position: 'relative', overflow: 'hidden'
              }}>
                {log.snapshot ? (
                  <img
                    src={`http://127.0.0.1:8000${log.snapshot}`}
                    alt="unknown face"
                    style={{ width: '100%', height: '100%', objectFit: 'cover' }}
                  />
                ) : (
                  <div style={{ textAlign: 'center', color: '#5A6A85' }}>
                    <div style={{ fontSize: '48px' }}>👤</div>
                  </div>
                )}
                <div style={{
                  position: 'absolute', top: '10px', left: '10px',
                  padding: '3px 10px', borderRadius: '20px',
                  background: 'rgba(246,79,59,0.8)', backdropFilter: 'blur(8px)',
                  fontSize: '10px', fontWeight: 700, color: 'white'
                }}>⚠ UNKNOWN</div>
              </div>

              <div style={{ padding: '14px 16px' }}>
                <div style={{ fontSize: '12px', color: '#F64F3B', fontWeight: 600, marginBottom: '4px' }}>
                  Unregistered Person
                </div>
                <div style={{ fontSize: '12px', color: '#5A6A85', marginBottom: '14px' }}>
                  🕐 {log.detected_at}
                </div>
                <Btn
                  onClick={() => setRegisterModal(log)}
                  style={{ width: '100%', padding: '8px', fontSize: '12px' }}
                >
                  ＋ Register This Person
                </Btn>
              </div>
            </div>
          ))}
        </div>
      )}

      {logs.length > 0 && (
        <div style={{
          marginTop: '24px', padding: '16px 20px',
          background: '#0D1421', border: '1px solid rgba(255,255,255,0.06)',
          borderRadius: '12px', display: 'flex', justifyContent: 'space-between', alignItems: 'center'
        }}>
          <div style={{ fontSize: '13px', color: '#5A6A85' }}>
            Showing {logs.length} unknown detections
          </div>
          <Btn variant="danger" onClick={handleClearAll} disabled={clearing} style={{ padding: '7px 16px', fontSize: '12px' }}>
            {clearing ? 'Clearing...' : '🗑 Clear All Logs'}
          </Btn>
        </div>
      )}

      {/* Register from snapshot modal */}
      {registerModal && (
        <RegisterFromSnapshotModal
          log={registerModal}
          onClose={() => setRegisterModal(null)}
          onSuccess={() => {
            setRegisterModal(null)
            fetchLogs()
          }}
        />
      )}
    </div>
  )
}

function RegisterFromSnapshotModal({ log, onClose, onSuccess }) {
  const webcamRef = useRef(null)
  const [departments, setDepartments] = useState([])
  const [captureMode, setCaptureMode] = useState('snapshot') // snapshot / webcam / upload
  const [capturedImage, setCapturedImage] = useState(null)
  const [uploadedFile, setUploadedFile] = useState(null)
  const [previewUrl, setPreviewUrl] = useState(
    log.snapshot ? `http://127.0.0.1:8000${log.snapshot}` : null
  )
  const [newDept, setNewDept] = useState('')
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState(null)
  const [form, setForm] = useState({
    name: '', department_id: '', designation: '', email: '', phone: ''
  })

  useEffect(() => {
    getDepartments().then(r => setDepartments(r.data))
  }, [])

  const handleCapture = useCallback(() => {
    const img = webcamRef.current.getScreenshot()
    setCapturedImage(img)
    setPreviewUrl(img)
  }, [])

  const handleFileUpload = (e) => {
    const file = e.target.files[0]
    if (!file) return
    setUploadedFile(file)
    setPreviewUrl(URL.createObjectURL(file))
  }

  const handleAddDept = async () => {
    if (!newDept.trim()) return
    try {
      const res = await fetch('http://127.0.0.1:8000/api/faculty/departments/', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${localStorage.getItem('access_token')}`
        },
        body: JSON.stringify({ name: newDept })
      })
      const data = await res.json()
      setDepartments(prev => [...prev, data])
      setNewDept('')
    } catch (e) { console.error(e) }
  }

  const handleSubmit = async () => {
    setError(null)
    if (!form.name || !form.department_id || !form.designation || !form.email || !form.phone) {
      setError('All fields are required')
      return
    }
    setLoading(true)
    try {
      const formData = new FormData()
      Object.entries(form).forEach(([k, v]) => formData.append(k, v))

      if (captureMode === 'snapshot' && log.snapshot) {
        // Fetch snapshot image and convert to blob
        const imgRes = await fetch(`http://127.0.0.1:8000${log.snapshot}`)
        const blob = await imgRes.blob()
        formData.append('image', blob, 'snapshot.jpg')
      } else if (captureMode === 'upload' && uploadedFile) {
        formData.append('image', uploadedFile)
      } else if (captureMode === 'webcam' && capturedImage) {
        formData.append('image_base64', capturedImage)
      } else {
        setError('Please select a face photo')
        setLoading(false)
        return
      }

      await registerFaculty(formData)
      onSuccess()
    } catch (e) {
      setError(e.response?.data?.error || 'Registration failed')
    } finally { setLoading(false) }
  }

  const inputStyle = {
    width: '100%', padding: '9px 12px',
    background: 'rgba(255,255,255,0.04)',
    border: '1px solid rgba(255,255,255,0.08)',
    borderRadius: '8px', color: '#F0F4FF',
    fontSize: '13px', outline: 'none', fontFamily: 'inherit'
  }

  const labelStyle = {
    fontSize: '11px', color: '#5A6A85',
    display: 'block', marginBottom: '5px',
    textTransform: 'uppercase', letterSpacing: '0.5px'
  }

  return (
    <div style={{
      position: 'fixed', inset: 0, zIndex: 1000,
      background: 'rgba(0,0,0,0.8)', backdropFilter: 'blur(6px)',
      display: 'flex', alignItems: 'center', justifyContent: 'center',
      padding: '20px'
    }}>
      <div style={{
        background: '#0D1421', border: '1px solid rgba(255,255,255,0.08)',
        borderRadius: '16px', width: '100%', maxWidth: '780px',
        maxHeight: '90vh', overflowY: 'auto'
      }}>
        {/* Header */}
        <div style={{
          padding: '20px 24px', borderBottom: '1px solid rgba(255,255,255,0.06)',
          display: 'flex', justifyContent: 'space-between', alignItems: 'center'
        }}>
          <div>
            <div style={{ fontSize: '16px', fontWeight: 700 }}>Register Unknown Person</div>
            <div style={{ fontSize: '12px', color: '#5A6A85', marginTop: '2px' }}>
              Detected at {log.detected_at}
            </div>
          </div>
          <button onClick={onClose} style={{
            background: 'transparent', border: 'none', color: '#5A6A85',
            cursor: 'pointer', fontSize: '20px', padding: '4px'
          }}>✕</button>
        </div>

        <div style={{ padding: '24px', display: 'grid', gridTemplateColumns: '1fr 280px', gap: '24px' }}>

          {/* Left — Form */}
          <div>
            <div style={{ marginBottom: '12px' }}>
              <label style={labelStyle}>Full Name</label>
              <input placeholder="Dr. Ramesh Kumar" value={form.name}
                onChange={e => setForm({ ...form, name: e.target.value })}
                style={inputStyle} />
            </div>

            <div style={{ marginBottom: '12px' }}>
              <label style={labelStyle}>Department</label>
              <select value={form.department_id}
                onChange={e => setForm({ ...form, department_id: e.target.value })}
                style={{ ...inputStyle, color: form.department_id ? '#F0F4FF' : '#5A6A85' }}>
                <option value="">Select Department</option>
                {departments.map(d => (
                  <option key={d.id} value={d.id} style={{ background: '#0D1421' }}>{d.name}</option>
                ))}
              </select>
              <div style={{ display: 'flex', gap: '6px', marginTop: '6px' }}>
                <input placeholder="Add new department..."
                  value={newDept} onChange={e => setNewDept(e.target.value)}
                  onKeyDown={e => e.key === 'Enter' && handleAddDept()}
                  style={{ ...inputStyle, fontSize: '12px', padding: '7px 10px' }} />
                <Btn variant="ghost" onClick={handleAddDept}
                  style={{ padding: '7px 12px', fontSize: '12px', whiteSpace: 'nowrap' }}>
                  Add
                </Btn>
              </div>
            </div>

            <div style={{ marginBottom: '12px' }}>
              <label style={labelStyle}>Designation</label>
              <select value={form.designation}
                onChange={e => setForm({ ...form, designation: e.target.value })}
                style={{ ...inputStyle, color: form.designation ? '#F0F4FF' : '#5A6A85' }}>
                <option value="">Select Designation</option>
                {['Professor', 'Associate Professor', 'Assistant Professor', 'HOD', 'Lab Assistant', 'Lecturer'].map(d => (
                  <option key={d} value={d} style={{ background: '#0D1421' }}>{d}</option>
                ))}
              </select>
            </div>

            <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: '10px', marginBottom: '12px' }}>
              <div>
                <label style={labelStyle}>Email</label>
                <input type="email" placeholder="name@bit.edu.in"
                  value={form.email} onChange={e => setForm({ ...form, email: e.target.value })}
                  style={inputStyle} />
              </div>
              <div>
                <label style={labelStyle}>Phone</label>
                <input type="tel" placeholder="9876543210"
                  value={form.phone} onChange={e => setForm({ ...form, phone: e.target.value })}
                  style={inputStyle} />
              </div>
            </div>

            {error && (
              <div style={{
                padding: '10px 12px', borderRadius: '8px', marginBottom: '12px',
                background: 'rgba(246,79,59,0.1)', border: '1px solid rgba(246,79,59,0.2)',
                fontSize: '13px', color: '#F64F3B'
              }}>{error}</div>
            )}

            <div style={{ display: 'flex', gap: '10px' }}>
              <Btn variant="ghost" onClick={onClose} style={{ flex: 1 }}>Cancel</Btn>
              <Btn onClick={handleSubmit} disabled={loading} style={{ flex: 1 }}>
                {loading ? 'Registering...' : 'Register Faculty'}
              </Btn>
            </div>
          </div>

          {/* Right — Photo */}
          <div>
            <div style={{ fontSize: '11px', color: '#5A6A85', marginBottom: '10px', textTransform: 'uppercase', letterSpacing: '0.5px' }}>
              Face Photo
            </div>

            {/* Mode tabs */}
            <div style={{ display: 'flex', gap: '4px', marginBottom: '12px' }}>
              {[
                { key: 'snapshot', label: '📸 Snapshot' },
                { key: 'webcam', label: '📷 Webcam' },
                { key: 'upload', label: '📁 Upload' }
              ].map(m => (
                <button key={m.key}
                  onClick={() => {
                    setCaptureMode(m.key)
                    if (m.key === 'snapshot') {
                      setPreviewUrl(log.snapshot ? `http://127.0.0.1:8000${log.snapshot}` : null)
                      setCapturedImage(null)
                      setUploadedFile(null)
                    } else {
                      setPreviewUrl(null)
                      setCapturedImage(null)
                      setUploadedFile(null)
                    }
                  }}
                  style={{
                    flex: 1, padding: '7px 4px', borderRadius: '6px',
                    cursor: 'pointer', border: '1px solid', fontFamily: 'inherit',
                    fontSize: '11px', fontWeight: 500,
                    background: captureMode === m.key ? 'rgba(59,123,246,0.12)' : 'transparent',
                    borderColor: captureMode === m.key ? 'rgba(59,123,246,0.4)' : 'rgba(255,255,255,0.08)',
                    color: captureMode === m.key ? '#3B7BF6' : '#5A6A85'
                  }}>
                  {m.label}
                </button>
              ))}
            </div>

            {/* Preview box */}
            <div style={{
              width: '100%', aspectRatio: '1', borderRadius: '10px',
              overflow: 'hidden', marginBottom: '10px', background: '#050810',
              border: '1px solid rgba(255,255,255,0.06)',
              display: 'flex', alignItems: 'center', justifyContent: 'center'
            }}>
              {captureMode === 'webcam' && !capturedImage && (
                <Webcam ref={webcamRef} screenshotFormat="image/jpeg"
                  style={{ width: '100%', height: '100%', objectFit: 'cover' }} />
              )}
              {previewUrl && (
                <img src={previewUrl} alt="preview"
                  style={{ width: '100%', height: '100%', objectFit: 'cover' }} />
              )}
              {!previewUrl && captureMode !== 'webcam' && (
                <div style={{ textAlign: 'center', color: '#5A6A85' }}>
                  <div style={{ fontSize: '36px' }}>👤</div>
                </div>
              )}
            </div>

            {/* Actions */}
            {captureMode === 'webcam' && (
              !capturedImage
                ? <Btn onClick={handleCapture} style={{ width: '100%', fontSize: '12px', padding: '8px' }}>
                    📷 Capture
                  </Btn>
                : <Btn variant="ghost" onClick={() => { setCapturedImage(null); setPreviewUrl(null) }}
                    style={{ width: '100%', fontSize: '12px', padding: '8px' }}>
                    🔄 Retake
                  </Btn>
            )}

            {captureMode === 'upload' && (
              <>
                <input type="file" accept="image/*" onChange={handleFileUpload}
                  id="unknown-upload" style={{ display: 'none' }} />
                <label htmlFor="unknown-upload">
                  <div style={{
                    padding: '8px', borderRadius: '8px', textAlign: 'center', cursor: 'pointer',
                    background: 'rgba(255,255,255,0.04)', border: '1px solid rgba(255,255,255,0.08)',
                    color: '#F0F4FF', fontSize: '12px'
                  }}>
                    {uploadedFile ? `✅ ${uploadedFile.name}` : '📁 Browse'}
                  </div>
                </label>
              </>
            )}

            {captureMode === 'snapshot' && log.snapshot && (
              <div style={{
                padding: '8px 10px', borderRadius: '8px',
                background: 'rgba(0,229,195,0.06)', border: '1px solid rgba(0,229,195,0.15)',
                fontSize: '11px', color: '#00E5C3', textAlign: 'center'
              }}>
                ✅ Using detected snapshot
              </div>
            )}
          </div>
        </div>
      </div>
    </div>
  )
}
