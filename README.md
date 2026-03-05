import axios from 'axios'

const API = axios.create({
  baseURL: 'http://127.0.0.1:8000/api'
})

API.interceptors.request.use(config => {
  const token = localStorage.getItem('access_token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// Auth
export const login = (data) => API.post('/auth/login/', data)

// Faculty
export const getFacultyList = (params) => API.get('/faculty/', { params })
export const getFacultyDetail = (id) => API.get(`/faculty/${id}/`)
export const registerFaculty = (data) => API.post('/faculty/register/', data)
export const reregisterFace = (id, data) => API.post(`/faculty/${id}/reregister/`, data)
export const getDepartments = () => API.get('/faculty/departments/')
export const deleteFaculty = (id, password) => API.post(`/faculty/${id}/delete/`, { password })

// Attendance
export const getTodayAttendance = () => API.get('/attendance/today/')
export const getAttendanceReport = (params) => API.get('/attendance/report/', { params })
export const getDepartmentSummary = () => API.get('/attendance/departments/')
export const getUnknownFaces = () => API.get('/attendance/unknown/')
export const deleteUnknownFace = (id) => API.delete(`/attendance/unknown/${id}/delete/`)
export const clearUnknownFaces = () => API.delete('/attendance/unknown/clear/')

// Recognition
export const recognizeFace = (imageBase64) =>
  API.post('/recognition/recognize/', { image: imageBase64 })

// Reports
export const exportCSV = (params) =>
  API.get('/reports/export/csv/', { params, responseType: 'blob' })

export default API
