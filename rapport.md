## Labo Load Balancing 

Authors: Cuénoud Robin, Dupont Maxime, Mulhauser Florian

## Introduction

The goal of this lab is to experiment with a web-application and a load balancer and analyze different implementation of the load balancer. We got familiar with the tools used for this lab, even though the windows fix caused some unforseen troubles (turns out git considers .png files as text files so we lost all our pictures, we had to had another line to .gitattribute and run some scripts to restore images we didn't have locally anymore). Then we implemented sticky session and looked at some modes offered by the load balancer.


## Pre-Task 1

I ran into some problems with the docker-compose ,   
I managed to fix them with the fix for windows after a while, I don't know why it wasn't working at first
or why it started working, my best guess is that I didn't create the `.gitattributes` file right.

I had to use the ip address `http://192.168.99.101:80` to get access to the load balancer, because I am using windows for this lab, and I can't directly access the containers (that IP is the IP of my VM running docker).


## Task 1 : Install the tools


#### Deliverables:

1. Explain how the load balancer behaves when you open and refresh the URL http://192.168.42.42 in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.

> When I refresh my browser, I can see that my request is handled by one server then another. That means the load-balancer is alternating between the 2 servers. That leads me to believe that the load-balancer is using a roundrobin policy.   
About session management, we can see that the session id changes everytime we refresh : 
>
>First time for our example :
>![](assets/img/1.1.1.PNG)

>Now we refresh : 
![](assets/img/1.1.2.PNG)  
We can see that the session id isnt the same, which makes sense because we are on another server, let's refresh one more time and see what happens :
![](assets/img/1.1.3.PNG) 
So now we are back the the same server, but we can see that the session id isn't the same. Which means that the user has a new session with the same server, this isn't a problem as of now but it can become one depending on what we implement onto new servers. It is safe to say that as of right now, there is no session management (since we just create a new one everytime).



2. Explain what should be the correct behavior of the load balancer for session management.

> The load balancer should use sticky sessions, basically that mecanism is used by the load balancer to know if a session has been established between the client and a server beforehand, and thus the load balancer can route the client to the same server it has a session with.

3. Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. Here is an example:



> ![](assets/img/1.3.png)



4. Provide a screenshot of the summary report from JMeter.


> ![](assets/img/1.4.PNG)


5. Run the following command:

`$ docker stop s1`

Clear the results in JMeter and re-run the test plan. Explain what is happening when only one node remains active. Provide another sequence diagram using the same model as the previous one.

> Now what's happening is that the load balancer routes the requests to the only server that's working, s2, so since it already knows the session ID , and can keep the same session with one user. Thus sessionViews is correctly incremented (because we stay inside the same session instead of creating a new one everytime).
![](assets/img/1.5.1.PNG)  
> We can see the JMeter results confirm the same thing we thought, every request is going to the same server. 
![](assets/img/1.5.2.PNG)  
>Here is a new graph showing what happens now that s1 is down :
![](assets/img/1.5.3.png)  
> Behaviour is mostly the same, except for the fact that we stay in the same session.

## Task 2: Sticky sessions

##### Deliverables:

1. There is different way to implement the sticky session. One possibility is to use the SERVERID provided by HAProxy. Another way is to use the NODESESSID provided by the application. Briefly explain the difference between both approaches (provide a sequence diagram with cookies to show the difference).

   > SERVERID is a server level id cookie, NODESESSID is an application level id cookie. The SERVERID is set by the HAProxy, the NODESESSID is set by the application. 
   >
   > In the first request the HAProxy set the `SERVERID` cookie to `1` (which stands for S1) and the S1 set the `NODESESSID` cookie. 
   >
   > In the second request in the following diagram show the HAProxy redirecting to S1 with the SERVERID cookie. An other user could have `SERVERID=2` and be redirected to S2.
   >
   > Note: Only the cookie that matter for the corresponding arrow are written.
   >
   > The `NODESESSID` can be unknow to the HAProxy. Therefore we'll use a `SERVERID` load balancing. 
   >
   > ![Artboard 1](assets/img/SHEMA01.png)
* Choose one of the both stickiness approach for the next tasks.  

2. Provide the modified haproxy.cfg file with a short explanation of the modifications you did to enable sticky session management.

> We decided to go for the `SERVERID` implementation, here are the modifications we did in the `haproxy.cfg` file : 

```
backend nodes
[...]

cookie SERVERID insert indirect nocache

# Define the list of nodes to be in the balancing mechanism
# http://cbonte.github.io/haproxy-dconv/2.2/configuration.html#4-server
server s1 ${WEBAPP_1_IP}:3000 check cookie s1
server s2 ${WEBAPP_2_IP}:3000 check cookie s2
```


> The first line instructs HAProxy to setup a cookie, only in the event that the client did not already provide one, this makes it so we have a cookie to use during load balancing. Then we added the `check cookie` line to allow the load balancer to know where to route requests based on the cookies.



3. Explain what is the behavior when you open and refresh the URL http://192.168.42.42 in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.


> ![](assets/img/2.3.PNG)  
> We can see here that the session stays the same, and the sessionViews get incremented, which means we enter the same session everytime. Since both servers are active, it means we implemented sticky sessions correctly (otherwise we would see a behavior similar to the one we had in task 1).  
> When we inspect the request, we can see our cookie is in there, and it is used by the load balancer to determine where to route it :  
>![](assets/img/2.3.2.PNG)





4. Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. We also want to see what is happening when a second browser is used.

   > ![image-20201202114904710](assets/img/2.4.4.png)
   >
   > When a second user is used, the HAPRoxy see's no `SERVERID` cookie so it uses round robin to determine the server. For example S2 and here's the diagram :
   >
   > ![2.4.2](assets/img/2.4.2.png)
   >
   > On the second request, the HAProxy identify the `SERVERID` cookie and send it to the correct server (S2 in this case). 

5. Provide a screenshot of JMeter's summary report. Is there a difference with this run and the run of Task 1?

>  ![](assets/img/2.5.PNG)  
We can see that we go to the same server everytime, which is different from task 1 (where it was a 50 / 50 split), that means we implemented sticky sessions correctly.


* Clear the results in JMeter.

* Now, update the JMeter script. Go in the HTTP Cookie Manager and uncheckverify that the box Clear cookies each iteration? is unchecked.

* Go in Thread Group and update the Number of threads. Set the value to 2.

7. Provide a screenshot of JMeter's summary report. Give a short explanation of what the load balancer is doing.

> ![](assets/img/2.7.PNG)  
We can see that we still have a 50/50 split, however it is for another reason, when thread1 (user1) got sent to a server , it got given a cookie, and then for everytime it requested again, it got sent to the same server, same as thread2, which was send to the other server because of the round robin policy, thus we have a 50/50 split  of which serverse were reached.


## Task 3: Drain mode

**Deliverables:**

1. Take a screenshot of the Step 5 and tell us which node is answering.

> We can see that the node s2 is answering (because we did multiple requests and only s2 is answering we also know that our sticky sessions are implemented correctly) : 
![](assets/img/3.1.PNG)


2. Based on your previous answer, set the node in DRAIN mode. Take a
    screenshot of the HAProxy state page.
    
    >  ![](assets/img/3.2.PNG)  
    We can see that the node is now in DRAIN mode.


3. Refresh your browser and explain what is happening. Tell us if you
    stay on the same node or not. If yes, why? If no, why? 
    
    > We stay on the same node when we refresh, this is because Drain mode is to prevent new connections, old sessions however, are preserved and still go to that node.

4. Open another browser and open `http://192.168.42.42`. What is
    happening?
    
    > When I open an incognito tab on Chrome or when I open another browser, I am redirected to s1, this is because the cookies aren't the same, and since s2 is in drain mode, the only server up accepting new connections is s1, so every request is send there by the load balancer.

5. Clear the cookies on the new browser and repeat these two steps
    multiple times. What is happening? Are you reaching the node in
    DRAIN mode?
    
    > I already kinda answered this question in the previous part, because I used more than 1 new browser, I am never reaching the node in DRAIN mode because each connection counts as a new one (we cleared our cookies), and Drain mode makes it so the node doesn't accept new connections, thus I am send towards s1 everytime.

6. Reset the node in READY mode. Repeat the three previous steps and
    explain what is happening. Provide a screenshot of HAProxy's stats
    page.
    
    > ![](assets/img/3.6.1.PNG)  
    This is the stat page after I set s2 in READY mode, time to test things out :
    
    
    > Everytime I clear my cookies I get sent to a different server, this is correct, because it counts as a new session so sticky sessions don't apply, and we just get sent to a server following the round robin policy.


7. Finally, set the node in MAINT mode. Redo the three same steps and
    explain what is happening. Provide a screenshot of HAProxy's stats
    page.
    
    > ![](assets/img/3.7.1.PNG)      
    We can see the state was correctly switched to maint on the second server.  
    
    > Now , whatever the connection I try, I always get send to s1, wheter it be a new broswer with cleared cookies, or my old browser which had a session with s2, everytime I just get a new session on s1, this is because s2 is in maintenance mode, which means that the server accepts no connections, regardless of sessions.


## Task 4: 

## Task 5:

1. `leastconn `seams effective because the server with the lowest number of connections receives it. 

   The second will be `uri` because since our web app got only one url it will be always the same so we can highlight a problem of this algorithm (for small application). 

## Conclusion

We have learned few ways of implementing an HAProxy and compared some ways to do it by running JMETER to compare them. This made the behaviour of a load balancer clearer in our mind, and introduced us to some ways git works, even though it wasn't the goal of this lab.