from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from .models import AttendanceRecord, UnknownFaceLog
from users.models import Faculty, Department
from django.utils import timezone
import os


@api_view(['GET'])
@permission_classes([IsAuthenticated])
def today_attendance(request):
    today = timezone.now().date()
    records = AttendanceRecord.objects.filter(
        date=today
    ).select_related('faculty', 'faculty__department').order_by('check_in_time')

    data = [{
        'faculty_name': r.faculty.name,
        'employee_id': r.faculty.employee_id,
        'department': r.faculty.department.name if r.faculty.department else '',
        'check_in_time': r.check_in_time.strftime('%I:%M %p'),
        'status': r.status,
        'photo': r.faculty.photo.url if r.faculty.photo else None
    } for r in records]

    total = Faculty.objects.filter(is_active=True).count()
    present = len(data)
    absent = max(0, total - present)

    return Response({
        'date': today.strftime('%d %B %Y'),
        'total_faculty': total,
        'present': present,
        'absent': absent,
        'records': data
    })


@api_view(['GET'])
@permission_classes([IsAuthenticated])
def attendance_report(request):
    date_from = request.query_params.get('date_from')
    date_to = request.query_params.get('date_to')
    department_id = request.query_params.get('department_id')
    status_filter = request.query_params.get('status')

    records = AttendanceRecord.objects.select_related(
        'faculty', 'faculty__department'
    ).order_by('-date', 'check_in_time')

    if date_from:
        records = records.filter(date__gte=date_from)
    if date_to:
        records = records.filter(date__lte=date_to)
    if department_id:
        records = records.filter(faculty__department_id=department_id)
    if status_filter:
        records = records.filter(status=status_filter)

    data = [{
        'faculty_name': r.faculty.name,
        'employee_id': r.faculty.employee_id,
        'department': r.faculty.department.name if r.faculty.department else '',
        'date': r.date.strftime('%d %b %Y'),
        'check_in_time': r.check_in_time.strftime('%I:%M %p'),
        'status': r.status,
    } for r in records]

    return Response(data)


@api_view(['GET'])
@permission_classes([IsAuthenticated])
def department_summary(request):
    today = timezone.now().date()
    departments = Department.objects.all()

    data = []
    for dept in departments:
        total = Faculty.objects.filter(department=dept, is_active=True).count()
        present = AttendanceRecord.objects.filter(
            faculty__department=dept,
            date=today
        ).count()
        absent = max(0, total - present)
        data.append({
            'department': dept.name,
            'total': total,
            'present': present,
            'absent': absent,
            'percentage': round((present / total * 100), 1) if total > 0 else 0
        })

    return Response(data)


@api_view(['GET'])
@permission_classes([IsAuthenticated])
def unknown_faces(request):
    logs = UnknownFaceLog.objects.order_by('-detected_at')[:50]
    data = [{
        'id': str(log.id),
        'detected_at': log.detected_at.strftime('%d %b %Y %I:%M %p'),
        'snapshot': log.snapshot.url if log.snapshot else None
    } for log in logs]
    return Response(data)


@api_view(['DELETE'])
@permission_classes([IsAuthenticated])
def delete_unknown_face(request, log_id):
    """Delete a single unknown face log and its image file"""
    try:
        log = UnknownFaceLog.objects.get(id=log_id)
        if log.snapshot:
            try:
                if os.path.isfile(log.snapshot.path):
                    os.remove(log.snapshot.path)
            except Exception:
                pass
        log.delete()
        return Response({'message': 'Deleted successfully'})
    except UnknownFaceLog.DoesNotExist:
        return Response({'error': 'Not found'}, status=404)


@api_view(['DELETE'])
@permission_classes([IsAuthenticated])
def clear_unknown_faces(request):
    """Delete all unknown face logs and their image files"""
    logs = UnknownFaceLog.objects.all()
    for log in logs:
        if log.snapshot:
            try:
                if os.path.isfile(log.snapshot.path):
                    os.remove(log.snapshot.path)
            except Exception:
                pass
    logs.delete()
    return Response({'message': 'Cleared successfully'})
