# Emobilis-payment-second
This is a program written in Django which facilitates payment by use of USSD
# 1. Creating Django environment
django-admin startproject payment_project
cd payment_project
django-admin startapp payment_app
# 2. Installing Django request
pip install django requests
# 3. Stating the payment  Model
from django.db import models

class Payment(models.Model):
    phone_number = models.CharField(max_length=15)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    reference = models.CharField(max_length=100)
    status = models.CharField(max_length=50, choices=[('Pending', 'Pending'), ('Completed', 'Completed')])
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.phone_number} - {self.amount}"
 # 4. Running migration
 python manage.py makemigrations
 python manage.py migrate
 # 5. pLacing CRUD operations
 from django.http import HttpResponse
from .models import Payment

def ussd_interface(request):
    session_id = request.GET.get('sessionId')
    phone_number = request.GET.get('phoneNumber')
    text = request.GET.get('text', '')

    response = ""
    text_array = text.split('*')

    if text == "":
        # Main Menu
        response = "CON Welcome to Payment Service\n"
        response += "1. Create Payment\n"
        response += "2. View Payment\n"
        response += "3. Update Payment\n"
        response += "4. Delete Payment\n"
    elif text_array[0] == "1":
        if len(text_array) == 1:
            response = "CON Enter amount:\n"
        elif len(text_array) == 2:
            response = "CON Enter reference:\n"
        elif len(text_array) == 3:
            amount = text_array[1]
            reference = text_array[2]
            Payment.objects.create(phone_number=phone_number, amount=amount, reference=reference, status="Pending")
            response = "END Payment created successfully."
    elif text_array[0] == "2":
        payments = Payment.objects.filter(phone_number=phone_number)
        if payments.exists():
            response = "CON Your Payments:\n"
            for payment in payments:
                response += f"{payment.id}. {payment.reference} - {payment.amount}\n"
            response += "0. Back\n"
        else:
            response = "END No payments found."
    elif text_array[0] == "3":
        if len(text_array) == 1:
            response = "CON Enter Payment ID to update:\n"
        elif len(text_array) == 2:
            payment_id = text_array[1]
            try:
                payment = Payment.objects.get(id=payment_id, phone_number=phone_number)
                response = f"CON Update {payment.reference}:\n1. Mark as Completed\n2. Cancel\n"
            except Payment.DoesNotExist:
                response = "END Payment not found."
        elif len(text_array) == 3:
            payment_id = text_array[1]
            option = text_array[2]
            try:
                payment = Payment.objects.get(id=payment_id, phone_number=phone_number)
                if option == "1":
                    payment.status = "Completed"
                    payment.save()
                    response = "END Payment marked as completed."
                elif option == "2":
                    response = "END Update canceled."
                else:
                    response = "END Invalid option."
            except Payment.DoesNotExist:
                response = "END Payment not found."
    elif text_array[0] == "4":
        if len(text_array) == 1:
            response = "CON Enter Payment ID to delete:\n"
        elif len(text_array) == 2:
            payment_id = text_array[1]
            try:
                payment = Payment.objects.get(id=payment_id, phone_number=phone_number)
                payment.delete()
                response = "END Payment deleted successfully."
            except Payment.DoesNotExist:
                response = "END Payment not found."
    else:
        response = "END Invalid input."

    return HttpResponse(response, content_type="text/plain")
# 6. Initiating URL

from django.urls import path
from . import views

urlpatterns = [
    path('ussd/', views.ussd_interface, name='ussd'),
]
# 7. Placing URL to the main project
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('payment_app.urls')),
]

