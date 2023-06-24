# security-compliance
How to check if a given port(s) is open to the internet and remediate it automatically ? 

Verifying, managing and automating compliance of resources on the cloud is a long and ongoing journey. When need to verify complaince of your existing aws resources and remediate them, if found non-compliant, the definition of compliance and the way it is remediated can be unique to the given organization.  
While there are growing number of security services [0] which can help you navigate this maze at scale, it needs significant effort for an oraganization to choose the best subset from this suite and operationalize it. 

While you might never encounter the exact usecase which the solution in this article is written to serve, it will still help you design your own compliance standard, with aws config rules [1], for a given resource and define your own remediation technique, with ssm automation documents [3]. 

This solution is written to: 
A. Check if any security group has an ingress rule open to 0.0.0.0/0 on a user-defined port(s).
b. Auto-remediate them either based on schedule or manual trigger or a security group update. 

The solution is based on a custom config rule [1], backed by a lambda function, which verifies all security groups in a given account/region and marks them non-compliant if the port(s) of interest are open to the internet. 
Remediation action of the custom config rule is based on a python based SSM document/automation [3] which removes[4] the ingress rule(s) which led to non-compliance. 

Before we start, let's appreciate the existence of some other similar looking solutions which serve a different purpose altogether:
1. While there is an AWS managed config rule 'vpc-sg-open-only-to-authorized-ports' [5] already present which enables us to do compliance check of all security groups but it is based on an allow list (allows only given port(s), declares the rest as non-compliant and leave them un-remediated) as compared to this solution which is based on a block list (blocks given port(s) only). And the remediation of this AWS managed rule has to be written separately separately by the consumer. 
2. There is also this solution [6] but it remediates based on instance IDs and its tags, not the security groups and ports directly, and the port(s) are not parameterized. 
3. This solution [7] notifies you of change to security group and remediates it but is not port based and is only forward looking. It cannot be used for checking compliance of existing security groups. 
4. If the purpose is to only restrict port 22, then you could use the AWS managed config rule: restricted-ssh [8] and remediate with ssm automation document: AWS-DisableIncomingSSHOnPort22

Launch the solution:
You can launch this cloudformation stack to create the below depicted solution:
https://github.com/vgn-1/security-compliance/blob/main/cloudformation-templates/sg-chosen-ports-restriction.yaml
![image](https://github.com/vgn-1/security-compliance/assets/109327302/2c8a1e83-61f3-4452-9cd2-3da83e7f27ba)


There are two parameters of behavorial significance:
1. SourceInputParameters
Use this parameter to specify the port(s) you desire to restrict. Maintain the json structure as in its description ( 22, 80 and 3389 are only kept as defaults):
{"FocusOnPorts": "22,80,3389"}
2. RemediationMode
Toggle the remediation mode of the config rule. Use 'True' if you want the rule to trigger remediation action automatically. If you only want to view the compliance status and not remediate, keep it as 'False'.

Upon successful creation, the stack creates the below resources:
<img width="679" alt="image" src="https://github.com/vgn-1/security-compliance/assets/109327302/87bb705d-5fd7-4874-9056-49a4a5463d54">


Test the solution:
1. Update the stack with SourceInputParameters parameter changed to the value '{"FocusOnPorts": "1234,5678"}'. This is to limit the scope of testing to a security group and port of choice and avoid broader impact. 
![image](https://github.com/vgn-1/security-compliance/assets/109327302/45ef7fdb-565f-4cad-b8b1-706b0b16389f)

2. Create a security group for this testing and add the below rules:
   <img width="856" alt="image" src="https://github.com/vgn-1/security-compliance/assets/109327302/87850008-186a-4f09-98af-4f7f8e18a963">

3. Navigate to AWS Config service console and look at the rule which just got created/updated:
![image](https://github.com/vgn-1/security-compliance/assets/109327302/0d5b9797-c790-43ce-a73f-9d45165d081c)


4. Evaluate the compliance with the given rule. This evaluation would run automatically every time a security group gets updated in your account.  Every revaluation leads to one backend lambda execution per security group. 
![image](https://github.com/vgn-1/security-compliance/assets/109327302/2488d616-a5bb-46d8-a977-82d8551f4501)

5. The compliance output would show only 1 security group as non-compliant, which we created in step#2 above.
   ![image](https://github.com/vgn-1/security-compliance/assets/109327302/25d9635c-4c06-4259-a2c8-da7ac0aeb11e)


4. Once the remediation action is run, only the marked ingress rules should be deleted and the rest remain intact.
5. Run the remediation as below. This would run automatically once the cloudformation stack is updated with parameter RemediationMode=True. 
   ![image](https://github.com/vgn-1/security-compliance/assets/109327302/2cf670e3-4b1d-4972-ae78-8e4b39c0680b)

6. Witness that revised security group ingresses:

Summary:
It is important to understand that there are always multiple ways to do a given task and this is especially true on aws cloud platform. 
While the above demostrated how a custom rule and remediation can help ensure compliance of a given resource, it is always best to use an AWS managed rule, if one is available for your use case, as it saves the effort of writing the backend lambda code and even better if there is a Control Tower guardrail for the same as it saves the effort of deploying the solution into the whole organization at scale. 


Reference Articles:
[0] https://aws.amazon.com/products/security/
[1] https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config_develop-rules.html
[2] https://docs.aws.amazon.com/config/latest/developerguide/remediation.html
[3] https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html
[4] https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_RevokeSecurityGroupIngress.html
[5] https://docs.aws.amazon.com/config/latest/developerguide/vpc-sg-open-only-to-authorized-ports.html
[6] https://aws.amazon.com/blogs/security/how-to-auto-remediate-internet-accessible-ports-with-aws-config-and-aws-system-manager/
[7] https://aws.amazon.com/blogs/security/how-to-automatically-revert-and-receive-notifications-about-changes-to-your-amazon-vpc-security-groups/
[8] https://docs.aws.amazon.com/config/latest/developerguide/restricted-ssh.html


