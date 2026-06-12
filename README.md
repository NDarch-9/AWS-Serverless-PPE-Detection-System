# AWS Serverless PPE Detection System

An automated, cloud-native validation system designed to enforce safety compliance in high-risk environments such as construction sites, factories, and vital zones. By leveraging a serverless architecture, this solution acts as a preventive digital checkpoint that verifies workers' Personal Protective Equipment (PPE) before they enter the work zone, generating instant compliance reports for the Admin with zero server management and minimal operational cost.

---

## The Problem

In massive construction sites and industrial factories that house large workforces often exceeding 100 workers, a critical vulnerability lies at the entrance gates and during extended shifts.
* **Human Complacency:** Over long hours, workers and engineers tend to neglect wearing required protective gear (such as helmets and gloves) due to fatigue or lack of immediate oversight.
* **Limitations of Manual Supervision:** It is operationally impossible for safety officers to manually inspect every worker at entry points thoroughly without causing massive bottlenecks and delays.
* **Cost and Complexity Barriers:** Building traditional, heavy monitoring systems requires expensive local infrastructure, high maintenance, and constant human monitoring.

This led to a core engineering challenge: How can we monitor safety compliance in the simplest, most rigid, and most cost-effective way possible?

---

## The Solution

The solution is to transform safety verification into a strict, preventive digital checkpoint by connecting entrance validation directly to an AWS cloud infrastructure.

Instead of reactive monitoring inside the site after a violation occurs, the system operates on a preventive philosophy: checking the worker's compliance via an uploaded image before site entry. This approach fosters self-awareness and accountability among the workforce, as access to the site is strictly conditional upon passing the AI validation check. 

Built using fully managed cloud services, this system achieves proof of concept with minimal development effort. It analyzes the entrance image and dispatches a comprehensive, real-time status report directly to the Admin, allowing them to instantly approve entry or enforce compliance at the gate. It is designed to seamlessly integrate with automated smart cameras or dedicated web/mobile applications in future phases.

---

## Solution Technical Details

The architecture is engineered entirely on Serverless principles to ensure high availability, rapid response times, and absolute cost efficiency (utilizing a pay-as-you-go model with zero idle server costs):

* **Amazon S3:** Serves as the central ingestion point. The worker's image captured at the gate is uploaded here, immediately triggering the backend pipeline.
* **AWS Lambda (Python):** The event-driven core of the system. It detects the new upload in S3, orchestrates the data flow, invokes the computer vision model, and handles the reporting logic.
* **Amazon Rekognition:** The managed computer vision service utilized to analyze the images. It leverages specialized protective equipment detection APIs to identify specific PPE types (helmets, gloves, face covers) and measure compliance metrics.
* **Amazon SNS (Simple Notification Service):** The automated notification engine responsible for formatting and routing the instant compliance or violation logs directly to the Admin's endpoint.

---

## Stack Flow



<img width="1684" height="1844" alt="image" src="https://github.com/user-attachments/assets/0d0cdcc8-a574-4caa-885b-bae6fda66135" />


### Workflow Description:
1. **Upload:** A picture of the worker is taken at the checkpoint before entering the site and is uploaded to an **Amazon S3** bucket.
2. **Trigger:** The upload event automatically invokes the **AWS Lambda** function without any human intervention.
3. **AI Analysis:** The `Lambda` function passes the image object references to **Amazon Rekognition** to execute the specialized PPE detection model.
4. **Admin Notification:** Based on the AI results, `Lambda` evaluates compliance, compiles an official log, and commands **Amazon SNS** to dispatch an immediate report to the **Admin** outlining whether access should be granted or denied.

---

## Architecture Design

<img width="2428" height="1650" alt="Blank diagram (4)" src="https://github.com/user-attachments/assets/fab2a908-1a38-4b8f-a864-2dc4c599ae05" />


The engineering design focuses on maximum efficiency and architectural cleanliness:
* **Decoupled Backend Focus:** By eliminating the initial need for a complex frontend interface, the project prioritizes robust backend synchronization and cloud resource utilization. This drastically reduces development time while delivering an enterprise-ready API layer that admins can monitor via direct notifications.
* **Future-Proof Scalability:** The current serverless pipeline is natively designed to scale out. It can seamlessly transition from static gate image uploads to live, continuous video streaming analysis (via services like Amazon Kinesis) in future updates without rewriting the core infrastructure.

---

## Lambda Source Code

The following clean, modular **Python** script runs within the AWS Lambda environment utilizing the `Boto3` SDK to handle the end-to-end evaluation and admin reporting:

```python
import boto3

def lambda_handler(event, context):
    rekognition = boto3.client('rekognition')
    sns = boto3.client('sns')
   
    # Target SNS Topic ARN for Admin alerts
    SNS_TOPIC_ARN = "arn:aws:sns:us-east-1:115960787563:MyProjectAlerts"
   
    try:
        # Extract bucket and object key automatically from the S3 event metadata
        bucket_name = event['Records'][0]['s3']['bucket']['name']
        image_name = event['Records'][0]['s3']['object']['key']
       
        print(f"--- Processing compliance check for image: {image_name} ---")

        # Invoke the specialized PPE detection API
        response = rekognition.detect_protective_equipment(
            Image={'S3Object': {'Bucket': bucket_name, 'Name': image_name}},
            SummarizationAttributes={
                'MinConfidence': 65,
                'RequiredEquipmentTypes': ['HEAD_COVER', 'HAND_COVER', 'FACE_COVER']
            }
        )
       
        all_results = []

        # Parse detected persons and check specific body parts for required gear
        for person in response['Persons']:
            found_equipment = []
            for body_part in person['BodyParts']:
                for item in body_part['EquipmentDetections']:
                    found_equipment.append(item['Type'])
           
            found_set = set(found_equipment)
           
            # Enforce rigid safety logic sequence
            if 'HEAD_COVER' not in found_set:
                status = "CRITICAL: Worker is missing a safety helmet (HEAD_COVER). Access Denied."
            elif 'HAND_COVER' not in found_set:
                status = "WARNING: Helmet detected, but safety gloves (HAND_COVER) are missing."
            elif 'FACE_COVER' not in found_set:
                status = "WARNING: Helmet and gloves detected, but face protection (FACE_COVER) is missing."
            else:
                status = "COMPLIANT: Worker is wearing all required PPE (Helmet, Gloves, Face Cover). Access Approved."
           
            all_results.append(status)

        # Formulate the final administrative summary log
        final_summary = "\n".join(all_results)
        message_body = f"Digital Safety Checkpoint Report for Admin:\nUploaded Image: {image_name}\n\nCompliance Analysis:\n{final_summary}"
       
        # Publish the report to the Admin's notification topic
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Message=message_body,
            Subject=f"[Admin Report] Safety Verification - {image_name}"
        )
       
        print("Compliance report successfully compiled and sent to Admin.")
       
        return {
            "status": "success",
            "results": all_results
        }
```

### Analysis Logic Breakdown:
1. **Confidence Threshold:** The model enforces a strict confidence bar (Confidence >= 65%) to ensure high reliability in object detection and prevent false compliance logs from reaching the Admin.
2. **Hierarchical Severity:** The code prioritizes life-safety items, immediately flagging missing head protection as a critical violation, while generating tailored warning strings for other gear items to facilitate quick corrections at the gate.

---

## The Result

### 1. Full Process Demo Video
The following video demonstrates the entire end-to-end operational flow of the system, showcasing the image upload to Amazon S3, the real-time AWS Lambda execution, and how the computer vision model evaluates worker compliance at the gate.


https://github.com/user-attachments/assets/52702861-5333-423b-bc24-639619cf6e01


### 2. Admin Email Notification
Once an image is processed by the serverless pipeline, an instant notification is generated. Below is a screenshot of the official compliance report delivered directly to the Admin's inbox via Amazon SNS, providing explicit details about the worker's protective gear status.

![Admin Email Notification](YOUR_ADMIN_EMAIL_IMAGE_PATH_HERE)

---

## Conclusion

This project demonstrates that robust industrial safety solutions do not necessitate bloated on-premise hardware deployments. By strategically orchestrating managed services like AWS Serverless and Amazon Rekognition, we can build a highly scalable, preventive gatekeeper that protects human lives and maintains regulatory alignment. By shifting from reactive onsite tracking to a strict entrance filter verified directly by real-time Admin alerts, this solution provides an efficient, low-cost, and maintenance-free blueprint for modern cloud-driven safety automation.

---
*Developed in: 06/2026*
    except Exception as e:
        print(f"Technical error during analysis pipeline: {str(e)}")
        return {"status": "error"}
