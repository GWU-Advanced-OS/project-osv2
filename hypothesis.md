# Overview of the project
  Our group aims to study OSv Unikernel. We decide to analyze the architecture/design of the system, thus having a general overview of the system. Also, we focus on its security, to find out how it meets the security principles. Finally, we plan to find possible existing applications of OSv, in order to find its good practice. 
  Basically, our goal is to find out the advantages and disadvantages of OSv, and how its goals are achieved, thus we need to touch the implementation of the system. More questions might be listed here when we dive deeper into the system.  

# Question from our own
1.How does OSv implement system calls, thus avoiding traditional linux system call overheads? 
2.How does OSv manage memory becuase it only has a single memory space? 
3.How does OSv support multiple threads, or scheduling between threads? 
4.Give a explanation of the lock scheme of OSv.
5.How does OSv support network stack?
 
# Question from project requirement  
1. Summarize the project, what it is, what its goals are, and why it exists.

   OSv is a unikernel aiming to provide a lightweight runtime VM enviroment for cloud VM applications, thus gaining performance and being very convenient to deploy applications quickly. It uses single address model, and only selects necessary libraries for an application to run, to compose the system image. Thus, one OSv image can only run one specific application, with multiple threads supported.
 
   For example, you have a Web server application running in a cloud. At a moment, a huge number of requests come in, you have no choice but to deploy massive VM servers in order to consume those requests. OSv is helpful in this case compared with traditional Linux VM because it is very light and small, thus having a very quick boot time. Providers can quickly and massively depoly their new servers. Also, because there is only one applicaton in one OSv, the performance gains in this case. This means that the provider can use less money to provide basically the same service quality. 
     

2. What is the target domain of the system? Where is it valuable, and where is it not a good fit? These are all implemented in an engineering domain, thus are the product of trade-offs. No system solves all problems (despite the claims of marketing material).

   The target domain of OSv is unikernel and virtual machine. It is valuable for those applications like microservices that only need a small & lightweight runtime environment, not only improving the performance for those applications but also providing the ability to massively deploy services in a short time. Also, as an unikernel, because OSv is within a VM, it doesn't need to bother with different hardware drivers, the code base could be relative small, thus having less potential bugs.  

   The security of the system could also be guaranteed because each OSv instance is running under a VM, thus providing strong isolation.
  
   Disadvantages also exist in OSv. First, it is not easy to deploy complex systems within OSv because it only has a single address space and does not support multiple process. Thus, APIs like fork()/vfork()/exec() cannot be used. However, modern applications usually compose of different components, each of which is a single processes. Transplanting those kind of systems running in OSv is not easy. Second, because OSv is running in a VM, different components have to be put into different VMs/OSvs, thus communication between components is not flexible compared with Linux VM guest and probably causing poor performance for applications that heavily rely on IPC. 

3. What are the "modules" of the system (see early lectures), and how do they relate? Where are isolation boundaries present? How do the modules communicate with each other? What performance implications does this structure have?

4. What are the core abstractions that the system aims to provide. How and why do they depart from other systems?
In what conditions is the performance of the system "good" and in which is it "bad"? How does its performance compare to a Linux baseline (this discussion can be quantitative or qualitative)?

5. What are the core technologies involved, and how are they composed?

6. What are the security properties of the system? How does it adhere to the principles for secure system design? What is the reference monitor in the system, and how does it provide complete mediation, tamperproof-ness, and how does it argue trustworthiness?

7. What optimizations exist in the system? What are the "key operations" that the system treats as a fast-path that deserve optimization? How does it go about optimizing them?

Subjective: What do you like, and what don't you like about the system? What could be done better were you to re-design it?
