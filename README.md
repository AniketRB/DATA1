from django.urls import path
from . import views

urlpatterns = [
    path('today/', views.today_attendance, name='today_attendance'),
    path('report/', views.attendance_report, name='attendance_report'),
    path('departments/', views.department_summary, name='department_summary'),
    path('unknown/', views.unknown_faces, name='unknown_faces'),
    path('unknown/<uuid:log_id>/delete/', views.delete_unknown_face, name='delete_unknown_face'),
    path('unknown/clear/', views.clear_unknown_faces, name='clear_unknown_faces'),
]
