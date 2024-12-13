FROM python:3.10-slim AS base

# Set working directory
WORKDIR /app

# Copy application files
COPY . /app

# Install Flask
RUN pip install flask

# Expose application port
EXPOSE 5000

# Set default command
CMD ["python", "app.py"]