# Auto-scaling-real-time-face-recognition
An auto-scalable (up to 20 EC2 instances) distributed application that performs real-time image recognition utilizing Amazon PaaS service AWS Lambda and also Raspberry Pi-based IoT using a camera connected to the pi.


Problem statement
In this project, we built a distributed application that utilizes PaaS service of Amazon, IoT devices to perform live, real-time image recognition. Specifically, we have developed this scalable application using Amazon PaaS service AWS Lambda and also Raspberry Pi-based IoT using a camera connected to the pi. Lambda is the first and the most widely used Function as a Service PaaS. Raspberry Pi is the most widely used IoT development platform. 

Design and implementation
Architecture

In this project, we are using Raspberry Pi hardware for video recording and extracting frames, S3 buckets to store those frames as well as the video (0.5 sec duration), AWS Lambda with docker containers, Dynamo DB to store the academic information and SQS to maintain the order of the output and ensure its delivery.


2.1.1 Services used and logic in the project

Amazon S3 Buckets (Simple Storage Service): 

For this project, we are using standard S3 bucket for storing frames extracted from the 0.5 second videos took using the camera connected to raspberrypi. We are using 2 buckets, one to store the frames and one to store the actual videos we took. S3 helps in persisting the data hence we used this service in our project to persist the input data of videos.

AWS SQS (Simple Queue Service):

We are using standard SQS for outgoing messages from the lambda(response queue) in our architecture. SQS makes it easy to transfer any volume and any level of throughput without losing messages. SQS also helps decouple the whole application as the lambda and raspberrypi’s video capturing application can work without affecting each other. SQS stores messages until the messages are read by the application in the pi and then deleted after processing.

AWS Lambda

We are using the lambda function with docker containers as a runtime environment for the purpose of performing the image recognition process. We are using an S3 bucket to store the frames extracted from the video captured through raspberry pi. Each entry in the S3 bucket would trigger the lambda function, which inturn triggers the lambda function. The main functionality of the lambda function is to spin up a docker container and pull image frames from the S3 bucket and pass it as an argument to the image recognition python script “eval_face_recognition.py”. This will generate the image recognition result which will be further parsed to extract the student name. This student name will be used as a key to fetch the student details from dynamoDB, which is in JSON string format. The fetched details will be further pushed to the SQS queue in the same format for persistence (as a buffer) before it is being pulled by the script running in raspberry pi. The load balancing is done by auto-scaling container resources which is completely handled by the lambda function itself. At peak, it reached a maximum of 16 concurrent executions and at the lowest load, there were only 1 concurrent execution.

Dynamo DB

In this project, we use Amazon Dynamo DB (a NoSQL database service) to store the academic information such as id, name, major and year. We have added few of the mentioned information manually in the database. In the lambda function, after getting the result from the face-recognition, we fetch the other information for the particular name recognized (that also acts as the key) and store it in the form of json string. This value/string is appended to the SQS response queue to maintain the order.

Raspberry Pi

Raspberry Pi is basically a credit card sized computer that can be plugged into a monitor (using HDMI cable) and we can use a mouse and keyboard (using USB). There is a camera module mounted on the raspberry pi that we will use to capture videos. To work with Raspberry Pi, we needed to first install the Raspberry Pi OS from the website. 

Docker Image for lambda function

We built a docker image to integrate it with the lambda function, where a new docker container will be created for every lambda function trigger through S3. The docker image contains all necessary modules and packages installed for setting up a lambda runtime environment and to perform image recognition. The custom trained image recognition model is added to the image during build time. This custom trained model is used inside docker containers for real time student image recognition purposes. The image also contains code to perform functionalities such as pulling student details from dynamoDB, frames from S3, and pushing output results to the SQS queue. 

Concurrency and Latency

To achieve concurrency, the first step we took was to implement multithreading to process video recording and also extracting frames from each video and then uploading those frames and videos parallelly to S3. This also helped in getting a very good latency of around 3-5 seconds. The second stage of concurrency is implemented in the execution of lambda function where concurrent processes will be parallelly pulling images from S3 bucket and run image recognition algorithm on each one of them. After a parallel image recognition process, the lambda function will concurrently fetch student details from the dynamo DB and push the JSON format output to SQS standard output Queue. The parallel threads running in the raspberry pi pull output messages in the queue for every 0.5 seconds and display them in the terminal. This way of concurrent processing helped us reduce latency significantly when we tested end-end from the point of continuous video recording to the display of output result.

Testing and evaluation

We tested our application several times using the live videos captured from raspberrypi’s camera. 
We trained our model with each of the group members' face images, multiple times because of low real time accuracy. We made sure we are achieving good real time accuracy above 60% as required.
We made sure that frames are extracted from all the 0.5 second videos recorded on the pi. We aslo made checked for length of videos and overall duration of the videos to be 5 minutes.
We added our data to dynamoDD and made sure the data is valid JSON.
We checked the output we got from lambda and made sure the format is matching the requirement.
We checked S3 bucket if it is getting all the frames from the raspberrypi. We also checked the bucket that gets the videos. It was adding up to 5 minutes as required
We checked that lambda is indeed running concurrently that is invocation more than 3 functions at any point of time.
Since we are using SQS to send output messages to raspberrypi, we made sure the SQS queue is getting all the output messages and aslo made sure they are not deleted until they are processed.
We also tested our multithreading approach whether all the threads are benign started and run properly.
We checked the outputs in raspberrypi and if they are matching the name of the person who is facing the camera. We kept retraining our model to get this real time accuracy high.

Code

raspberrypi
main.py: This is the entry point of our application. We are using python’s picamera module to capture the videos from the camera attached to the raspberry pi. Here we start capturing videos with duration of 0.5 seconds each and store them in a folder and then we initialize a thread for each video that works on frame extraction and uploading to S3. We also have the code here to accumulate all the threads running and wait for them to complete and not exit before the threads execution.

sendToS3.py:  Here, we have a function that takes input as video’s filename we get from the thread’s arguments passed from main.py. It then extracts the middle frame from the video and then uploads the frame to S3 with key as the filename with the number of the video (e.g clip0001.jpg). This function keeps checking for output we store in a dictionary from another file we explain below (pollSQS.py). It prints the latency and the output once the output for that particular frame has been detected in the dictionary.

pollSQS.py: Here, we have a function that polls the reqponse Amazon SQS for any new messages. If a new message is received, it appends that message into a dictionary with key as the name of the clip/frame that we sent to S3, so that our sendToS3 function can check for the corresponding output in the dictionary. It then deletes that particular message from SQS after storing it in the dictionary.

videosToS3.py:  This is another process that runs parallel to the main application. This is written to keep checking for new video files being created in a particular folder and then uploads them to S3 as and when created. This process is used to make sure the videos are persisted in S3 buckets.

To run these, we installed the following python libraries
boto3 (to work with AWS services from python)
picamera (to utilized the camera in raspberrypi)
glob2 (to work with file structure in python)
watchdog (to watch for new videos created in a folder)
We have to run the main.py script to start the application. The videosToS3 runs parallelly along with the main script. All the other services are internally managed using threads inside the main.py file itself.
Amazon access keys have to be setup in the ~./aws folder before using the application. also S3 bucket (videosToS3.py and sendToS3.py) names and SQS url( pollSQS.py)  have to be changed in the code to be able to successfully execute.

Docker Image

Dockerfile
This file is used to build the docker image locally before pushing it to the Elastic container registry of AWS. This file contains docker commands to perform multiple operations such as copying files from the local mac machine to the filesystem inside the image, installing necessary python libraries and modules, setting up an entry point which will be executed at the time of container startup.

handler.py
This file is the entry point for the lambda function execution where the “face_recognition_handler(event, context)” method will be invoked as soon as a frame gets uploaded to the S3 bucket. It also contains methods to perform functionalities such as invoking image recognition model, fetch student details from dynamo DB, push output result to SQS queue.

References
https://www.raspberrypi.com/software/
https://docs.aws.amazon.com/
https://www.serverless.com/aws-lambda
Lecture videos




