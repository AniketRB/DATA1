@api_view(['POST'])
@permission_classes([IsAuthenticated])
def reregister_face(request, faculty_id):
    try:
        faculty = Faculty.objects.get(id=faculty_id)
    except Faculty.DoesNotExist:
        return Response({'error': 'Faculty not found'}, status=404)

    image_file = request.FILES.get('image')
    image_base64 = request.data.get('image_base64')

    if image_file:
        file_bytes = np.frombuffer(image_file.read(), dtype=np.uint8)
        img = cv2.imdecode(file_bytes, cv2.IMREAD_COLOR)
        image_file.seek(0)
    elif image_base64:
        img = decode_base64_image(image_base64)
    else:
        return Response({'error': 'Image required'}, status=400)

    embedding = get_embedding(img)
    if embedding is None:
        return Response({'error': 'No face detected'}, status=400)

    # Update face embedding
    FaceEmbedding.objects.filter(faculty=faculty).delete()
    FaceEmbedding.objects.create(
        faculty=faculty,
        embedding=embedding_to_json(embedding)
    )

    # Also update profile photo
    if image_file:
        # Delete old photo
        if faculty.photo and os.path.isfile(faculty.photo.path):
            os.remove(faculty.photo.path)
        faculty.photo = image_file
    elif image_base64:
        b64_data = image_base64
        if ',' in b64_data:
            b64_data = b64_data.split(',')[1]
        new_photo = ContentFile(
            base64.b64decode(b64_data),
            name=f'{faculty.employee_id}_photo.jpg'
        )
        # Delete old photo
        if faculty.photo:
            try:
                if os.path.isfile(faculty.photo.path):
                    os.remove(faculty.photo.path)
            except Exception:
                pass
        faculty.photo = new_photo

    faculty.save()

    return Response({'message': 'Face and profile photo updated successfully'})
