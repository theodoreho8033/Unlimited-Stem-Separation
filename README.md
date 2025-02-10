# Unlimited-Stem-Separation


**Link to source code can be provided upon request. I'm not making it public due to security concerns. 

There are many stem separation tools that exist publicly online:  
- [Voice.ai Stem Splitter](https://voice.ai/tools/stem-splitter)  
- [BandLab Splitter](https://www.bandlab.com/splitter)  
- [FADR Stems](https://fadr.com/stems)

The main idea is to use ML algorithms to separate different instruments/sources from one track or audio file.

Most of these services contain paywalls and limit the use due to expensive GPU time for inference.

The goal of this project is to create a truly free and unlimited stem separation.

---

## How to Make It Free

To do this, two main approaches are used to reduce inference cost:

1. **Use a light-weight model that does not sacrifice quality.**

   I would guess most stem-splitters right now are using HT-Demucs, the most widely known stem separation algorithm.  
   Voice.ai is almost certainly using HT-Demucs because it is the only algorithm that offers guitar and piano as stems.  
   However, HT-Demucs is very large and expensive for inference. Recently a new algorithm was released that offers similar performance and drastically reduced inference time. See [SCNET](https://arxiv.org/pdf/2401.13276).  
   I am able to get a sub 10-second inference time with a 3-minute track on an A10 GPU with this algorithm.  
   For more information on the different models and their specific metrics, see [this document](https://github.com/ZFTurbo/Music-Source-Separation-Training/blob/main/docs/pretrained_models.md).

2. **Find a cost-effective GPU cloud service for deployment.**

   There are many cloud providers offering serverless GPU. My main concern is limiting the time spent on the GPU by optimizing cold-starts. This is difficult for AI inference applications because the CUDA dependencies required for the image are very heavy and ML images often run over 10GB.
   
   - **Cloud Run:** Offers a mounted container registry which allows for very fast cold-starts. However, you cannot configure the cooldown time for containers. So if the API is hit with low, sparse traffic, and each container starts up for only one request, you would need to pay for the 15-minute cooldown time as well. For a request which only takes ~10 seconds to run the inference, you would also be paying for 15 minutes of GPU for the cooldown.
   
   - **Lightning.AI:** Is great—you can configure the cooldown time of instances. However, there is one key problem. Lightning pulls your image from a registry that isn't cached. So for every cold-start you need to download the whole image. Even with optimizations such as lazy-pulling and docker-slim to reduce images, I still could not get the cold-start image pulling below ~40 seconds.
   
   - **Modal:** Finally, I settled on Modal. Modal has a very intelligent image indexing system and a super-optimized GPU function execution stack. They don't index on layers. Since a Docker container can be defined as a file structure, they index on the files themselves and have a FUSE mount to get the files as fast as possible. They can even snapshot the process memory for faster restoration. See this really cool talk to learn more: [Modal Talk](https://www.youtube.com/watch?v=3jJ1GhGkLY0).

   With Modal, from cold-start to inference to cooldown I am able to get a sub-20 second GPU time for a full 3-minute track—sometimes even sub-15 seconds. The main bottleneck is actually the data transfer of the audio. I was able to cut this down a bit by forcing the Modal instances to run on the same region. Particularly, US-East-1, the location of Modal's control plane. 

With these optimizations, I am able to get the inference cost per API request to a sub-$0.005 cost per request. For low traffic, this is definitely affordable. I'm hoping to get this down even more with optimizations to the data transfer, and potentially request free compute credits from Modal.

---

## How to Protect the API Endpoints

A main concern I have is that while it's free, if a malicious scripter tried to call my API I could incur heavy GPU costs. And I don't want users to need to create an account either. To do this, I require all users to have a session key before calling the API. This session key is only accessible through a persistent WebSocket connection that will be protected by CAPTCHA authentication. The key expires after the user completes the separation, if the WebSocket disconnects, or after a predetermined session timeout.

I still need to do some work for this. I have not implemented the session timeout, for example. I also might consult some of my system security friends to get their thoughts on this.

---

## What's Next?

After finalizing the backend, I will create the front-end and deploy.  
I plan to use this website as a loss leader to generate traffic for other cool music AI projects I have planned, which I will then monetize. I have some ideas I'm excited about involving a style transfer of music effects using differentiable digital signal processing—see [DDSP on GitHub](https://github.com/magenta/ddsp).
