So, What is Jenkins Job-DSL?
Jenkins is a wonderful system for managing builds, and people love using its UI to configure jobs. Job DSL was one of the first popular plugins for Jenkins which allows managing configuration as code.The Jenkins Job DSL enables the programmatic creation of Jenkins jobs using Groovy code. You can store this code in your Git repository and thus make changes traceable and generate Jenkins jobs automatically. The DSL is provided via the Job DSL Plugin and is documented in detail in the Job DSL API Viewer.

Agenda!!!

As Developer pushes the code in GitHub. So, It automatically deploys our full flesh Web Server ready. It’s not just a normal server. It’s a great setup which has fault tolerance capacity, Autoscaling capacity, and Monitoring also available.

Steps Involved
Create a Jenkins Seed job
From Git Push upload code in GitHub and Triggers Seed job.
Jenkins Dynamic cluster runs the Jenkins jobs.
Kubernetes launch the web server.
Prometheus monitor the things from getting metrics.
Grafana visualize the metrics properly.
Let’s Go
First, set up our lab. So, In single shot our whole Environment ready.

It’s a one-time setup you do. If you do this first time then maybe it’s a little bit painful but if you go through the right approach then its very simple.

First, Setup a Kubernetes. Here, I am using Minikube(Single-node cluster) for this. I install this on the top of Windows but the same configuration you can also do on Linux or Mac.

After install Minikube you need to monitor that. As because our web server launch inside it. For Monitoring here, I am using Prometheus and we need to collect the metrics of minikube. But, First I tell you it’s very painful for me to do this as because of nowhere available this thing on the google. For metrics collection, you need a node_exporter for Linux program because behind the scene minikube also based on Linux distribution. For this, you can go manually inside the minikube and download or through SSH you put this node_exporter on that. But, you can’t put this anywhere on the minikube.


Why???
But why???
Because After reboot maybe you not find your node exporter. Not every path is persistent in minikube. There are some limited paths which is persistent in minikube. I want to tell one them which is /data.

If you put anything inside the /data folder so it will be persistent in minikube. Now you put your software inside this and unzip this folder using the command :

tar -xzf node_exporter-1.0.1.linux-amd64.tar.gz
After that, you see a folder of the same name is generated there. Now, just come back from there. Now, you only need to just run the exporter program but you also face a great challenge as It’s not a good practise to go inside and run that program after every reboot. So what we do??? There is a path in minikube (/var/lib/boot2docker/bootlocal.sh). If you write any Script inside this file so it runs at the time of boot in minikube. Right now, Our need is to start the node_exporter after every reboot so we write the script to start this node exporter. Script is:

#!bin/bash
cd /data/node_exporter-1.0.0.linux-amd64;nohup ./node_exporter &
Now If you also want to monitor docker in minikube. But here is some challenge comes up. I will tell you. what challenge you face. So for getting metrics of docker we need to write/add the script in daemon file:

{
  "metrics-addr" : "0.0.0.0:9323",
  "experimental" : true
}
This on the docker daemon file(/etc/docker/daemon.json). But here, another challenge comes up. As minikube doesn’t store this data persistent on this file. After reboot, everything goes down. So, we need to make this file permanent. For keeping this file permanent there is a way in minikube. You can put this file on the $MINIKUBE_HOME/files. but it only syncs when minikube start is run, though before Kubernetes is started.

Like that we also need to put the above script file also in the same path. So after every reboot file will be present there. Like this


Minikube config
Now Your main setup is configured. You can start the minikube from command minikube start. After that, you can see the metrics of minikube in the browser. But you have not seen metics of docker. why???

Because our new daemon file is updated after docker services are started. To solve this issue there is no automated way is available. Believe me, I try every possible way. The only way to get the metrics of docker is to go inside the minikube and run command sudo systemctl restart docker .

Now, you can see the metrics of docker also. For see, the metrics need a Prometheus server. For setup Prometheus in Kubernetes, you can click here.

Prometheus Integrating With Grafana On The Top Of Kubernetes
when I talk about the DevOps world these two tools are very important for that …
medium.com

After that you see like this :


Metrics
Now, you also need your Jenkins Dynamic cluster be ready for this you can take reference from here. Here, I guide you how you can create your cluster in Jenkins.

CI/CD Pipeline on Jenkins Dynamic Clusters Integrating with Kubernetes
Hey all, Today I would like to show you a great advance setup for this Jenkins automation world …
www.linkedin.com

Finally, our setup is ready for work. Let’s quickly do the things.

Step 1:
Create a Jenkins Seed job which create our other Jenkins job and also create a pipeline.


Seed job

Seed job

Seed job

Seed Job
Here, you see for triggering this job I use two ways:

GitHub Hook Triggers: As when someone upload the code on GitHub repository so this job will triggers.
Trigger Build Remotely: As when someone wants to trigger this job so they can trigger through URL.
Step 2:
As Developer Commit the code from Git. So, it upload the code on GitHub and also triggers Seed job.

For doing this, I use Post-Hooks in Git


Post-hook
After creating post-commit in git. It automatically push and trigger the seed job as when you commit the code.

Step 3:
This is the main process step in our task. It create the whole setup of our web server. As developer commit the code and seed job runs. They create Jobs in Jenkins. In this step I show you the jobs which Jenkins runs.

Job 1

Download code from GitHub and make a docker image of our latest web server and also upload on docker registry. Intresting things is we create our jobs from Groovy language code.

job('job1') {
    description('Copy the code from github and also create and push an image in dockerHub')
    label('rhel')
    scm {
        git {                      
            remote {
                url('https://github.com/Harshetjain666/DevopsTask-6.git')
            }
            branch('*/' + 'master')
        }
    }
    steps{
        shell('mkdir /root/workspace ; cp -rf * /root/workspace/')
        dockerBuilderPublisher {
            dockerFileDirectory('/root/workspace/')
            cloud('')
            fromRegistry {
                credentialsId('')
                url('')
            }
            pushOnSuccess(true)
            cleanImages(false)
            cleanupWithJenkinsJobDelete(false)
            pushCredentialsId('424dca0c-5cf9-4bee-ae01-0d373301bf14')
            tagsString('harshetjain/html-environment')
        }    
    }
}
Job 2

This is very significant job. This made our web server ready and also do Autoscaling. So, as load increase behind the scene launch more servers.

job('job2') {
    description('Make Environment Ready And Updated')
    label('rhel')
    scm {
        git {
            remote {
                url('https://github.com/Harshetjain666/DevopsTask-6.git')
            }
            branch('*/' + 'master')
        }
    }
    triggers {
        upstream('job1', 'SUCCESS')
    }
    steps {
        shell('''mkdir /root/workspace ; cp -rf * /root/workspace/ 
            deployname=$(kubectl get deployment --selector=type=html --output=jsonpath={.items..metadata.name})
            if [ $deployname =="" ]
            then
            kubectl create -f /root/workspace/html.yml
            else
            echo "Environment updating ..."
            fi
            kubectl set image deployment  *=harshetjain/html-environment --selector=type=html --record''')
    }
}
Job 3

This job checks our web server is perfectly working or not. If they find any unstability so it contact to developer and also trigger other job to update the web server.

job('job3') {
    description('Test the webpage and send mail if job fails')
    label('rhel')
    triggers {
        upstream('job2', 'SUCCESS')
        pollSCM {
            scmpoll_spec(* * * * *)
        }
    }
    steps {
        shell('''status=$(curl -o /dev/null -s -w "%{http_code}" http://192.168.99.100:31000)
            if [ $status == 200 ]
            then 
            exit 0
            else 
            exit 1
            fi''')
    }
    publishers {
        mailer('hjain8620.hj@gmail.com', false, false)
        downstreamParameterized {
            trigger('job4') {
                condition('UNSTABLE')
                
            }
        }

    }
}
Job 4

This job is only run when if job 3 find our server not works good. If they find some issues in our updated web server so it automatically rollback our web server and change to the default one.

job('job4') {
    description('Run Over Old Setup')
    label('rhel')
    steps {
        shell('''kubectl set image deployment  *=harshetjain/html-environment:v1 --selector=type=html --record''')
    }
}
You can also create a Pipeline view so we can see what’s going on

buildPipelineView('Pipeline') {
    filterBuildQueue(true)
    filterExecutors(false)
    displayedBuilds(1)
    selectedJob('job1')
    alwaysAllowManualTrigger(true)
    showPipelineParameters()
    refreshFrequency(3)
}
Now we only need to start the job 1 for this use queue('job1')

Bang on!!! Your work is nearly completed.

Step 4:
You can see a very powerful and fault tolerance setup is ready in Kubernetes. If anything is goes down within a second new one is created with all configuration and same old data available.


Kubernetes info
Step 5:
I integrate both steps 5 and 6 in there as because for monitoring we need metrics and I also use here Grafana so we can visualize properly and also behind the scene Grafana get the metrics from Prometheus. For showing you this visualization I use a simple query process_cpu_seconds_total. This showing us the CPU utilization of servers.


Visualize
Now, At the end I would like to show you the web page of our web server. This is just a testing page which I made for testing.


Web page
This is Automation. Finally, you seen a very great CI/CD setup as if you again put the updated code on GitHub so within some seconds you see your server is updated. It’s not just only updates. Lots of things also done to make this setup very unique.
