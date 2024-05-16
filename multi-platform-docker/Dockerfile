# Use the specified base image
FROM public.ecr.aws/lambda/python:3.11-arm64

# Copy requirements.txt file to the container
COPY requirements.txt .

# Install the python requirements from requirements.txt
RUN python3.11 -m pip install -r requirements.txt 

RUN python3.11 -m pip cache purge

# Copy Python script into the container
COPY app.py .

EXPOSE 80

# Set the command to run the Python script
CMD ["app.lambda_handler"]
