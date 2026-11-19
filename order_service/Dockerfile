
# 1. Select Python base image
FROM python:3.12-slim

# 2. Set working directory
WORKDIR /app

# 3. Create a non-root user
RUN useradd -m appuser

# 4. Copy requirements file
COPY requirements.txt .

# 5. Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# 6. Switch to created user
USER appuser

# 7. Copy all application files
COPY --chown=appuser:appuser . .

# 8. Expose order service port
EXPOSE 8083

# 9. Start order application
CMD ["python", "order_app.py"]

