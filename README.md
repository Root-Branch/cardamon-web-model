# How does Cardamon Web work?

Cardamon Web is an SCI compliant, bottom-up model for estimating the power consumption and carbon emissions of web sites and web apps.

Let’s break this down a little. 

## What do we mean by SCI compliant?

The SCI or Software Carbon Intensity specification is an ISO standard for calculating the amount of carbon emissions released when using a piece of software. It is described using a set of high-level mathematical equations which can be applied to any kind of software.

We developed the Cardamon Web Model to explicitly implement the SCI spec and apply it directly to web-sites. You can see the Model’s SCI origins in the mathematics we use to describe it; the Cardamon Web Model is effectively “SCI for the web”.
What do we mean by bottom-up?
A bottom-up model is one that starts with individual components and builds up to the overall system that’s being modelled. This is in stark contrast to a top-down model which starts with high-level, system-wide data which is then broken down using statistics about the system in question.

Let’s compare these two approaches when trying to estimate the carbon emissions of a web-site. 

Using a top-down approach we would start by gathering economic and industry-wide data about the IT industry. For example, the estimated number of user devices and the energy used to power them. The estimated number of data-centres and the average energy they consume. The estimated total amount of data transferred over the internet etc. Once this data is gathered we can start to estimate the energy consumed by a single user device and the amount of power required for a single server within a data-centre. Whilst a top-down approach is quick and easy to calculate, in the case of web sites and web apps, starting from industry-wide data and zooming into a single user device accessing a single web page is likely to lead to large error-margins and overly generic insights into the individual web pages under examination. 

Cardamon Web takes a bottom-up approach. We first define clear subsystem boundaries for the hardware components that make up a website. For example, the user devices, networking infrastructure and web hosting (data centre). Then we identify the hardware components that contribute most to the power consumption of each subsystem e.g. CPU, network adapter, display etc

Each of these components is modelled individually using metrics gathered during operation and combined with data gathered from external tools such as user analytics platforms and cloud metrics tools. With sufficient data, we believe the bottom-up approach can achieve higher accuracy and provide more actionable insights into the inner workings of web sites than a top-down approach.

## Lets take a closer look at the Cardamon Web Model

Measuring a website consists of measuring one or more of its pages. Which pages you choose to measure, and how many pages you measure, depends on your website and what you are trying to achieve with the measurement.

For example, getting a measurement for an entire website may be simple if there are relatively few pages (a few hundred), it’s possible to measure every single page. However, doing the same for a website that contains many thousands or tens of thousands of pages is a little more difficult and statistical sampling may be required.

Once a set of pages has been established we load each web page on a physical reference device to simulate a user interacting with the page (details below). During this time we begin gathering metrics related to each component of the end user device. The purpose of this is to establish the energy consumption of a single hit to a single page. This can then be combined with user analytics (if available) to determine the likely energy consumption of the website over a given reporting period.

If geographical information is provided by the analytics platform we can also combine that with carbon intensity data in those locations providing an accurate picture of the carbon emissions produced in that time period.

Let’s take a look at each of the subsystems individually. 

### User Device (Frontend)

There are many different kinds of user devices on the market; mobiles, tablets, laptops, desktops etc. For practical reasons we model only two of these device types (mobile and desktop) although we plan to add more in the future.

A reference device for each device type is specified using market data and we build a physical computer to meet those specifications. Defining a reference removes the guess work from the model. We have an actual physical device that we can take detailed measurement from. 
We then divide its internal components into two categories: Dynamic components, those components whose energy consumption varies as the web page is interacted with. And static components, those components whose energy consumption remains mostly constant during web page interaction. Currently the dynamic components in end-user devices are the CPU, network adapter and screen. Other components - such as the GPU and RAM - will be added in future versions. All other components' are considered static and their power consumption makes up the device’s idle power.

For each dynamic component we define a mathematical function which converts relevant metrics into power consumption. These functions will be published in detail separately.

Now that we have reference devices we take each web page and load it on the device and begin interacting with it. The interaction is simple, we just load the page and begin scrolling to the bottom over a ten second window. Spending time on the page is important as it captures activity which occurs after first-load.

During this time we gather metrics; CPU utilisation, data transfers, colour profile of the screen etc. These metrics act as inputs into the functions used to model each component. We then add the results of these functions together along with the idle power consumption of the device to estimate the total energy consumption of the device during interaction.

### Network Infrastructure

Network infrastructure refers to all the telecom hubs, cables, wireless infrastructure, internet exchange points etc which enables data exchange over the internet.

This is the only part of the model that uses a top-down approach. It simply isn’t possible to measure the activity of the network infrastructure without access to the buildings and devices responsible for transferring data from one computer to another.

For this reason we resort to academic research which attempts to place a high-level estimate on the amount of energy required to transfer data over the internet. We use the numbers cited in Malmodin et al. 2020 (0.059 kWh / GB).

This portion of our model will improve with time and data and we are watching with anticipation the work and research undertaken by other initiatives specialising in network measurement, like Greening Of Streaming.

### Data Centre (Backend)

For the backend infrastructure we take a similar approach to the frontend. Using market data we define three reference servers (low end, mid range and high end). Each server is then broken down into dynamic and static components. Currently the dynamic components consist of the CPU and network adapter. Like the front end we will add to this list of components to include GPU, RAM etc in future versions. Every other component is considered a static component and collectively they make up the idle energy consumption of the server.

Unlike the frontend where it is assumed that a single device is being used to view the web page, it is possible that multiple servers or virtual servers are used in the backend. The servers may also be dedicated to the task of processing the website or - as is the case for virtual servers - they may be part of shared infrastructure which is hosting multiple unrelated things.

We model these two scenarios by calculating the number of virtual CPUs (vCPUs) each reference server can accommodate. We assume that the number of vCPUs is equal to the number of independent threads available to the physical CPU. If we know the number of vCPUs currently used to host a website, we can divide that number by the theoretical maximum number of vCPUs the reference device can support. This is the proportion of the server used to host the  website. All remaining vCPUs (the ones not used to host the website) are assumed to be taken by other tenants and utilized at 15% (the current estimated average datacentre utilization). This value is configurable if more information is known.

In the case of dedicated machines the calculation is much simpler: the number of vCPUs being used by your website is equal to the maximum number of vCPUs available and no other tenants are on the server.

Now that we have accounted for the percentage of each server responsible for hosting the website we can apply CPU utilization metrics and data transfer to calculate the energy consumption of the servers. As this data is likely to come from production metrics gathering tools such as a cloud providers dashboard this data will capture the operation of the website across all hits over a given time period. Any calculation based on this data will be mis-aligned with the calculations provided by the frontend portion of the model because the front end only includes those we pages that we measured and has been scaled by only those hits those pages received over the reporting period, not the total hits the website received over the reporting period. We need a final step to “align” these two figures.

This is achieved by knowing the total number of hits to the website. We know the number of hits covered by the frontend measurement; that came from the user analytics of the pages we measured. Dividing one by the other gives the ratio we need to either scale the frontend number up or the backend number down to bring these two numbers into alignment. If the total number of hits is unknown then we assume that the total number of hits the backend infrastructure received is equal to the number of hits the web pages we measured on the frontend received. In this case, the two numbers are already in alignment.

One final note is that the backend model has been developed so that cloud metrics data can be used. However, it’s possible to replace this data with real world measurements by instrumenting your servers with a tool such as Cardamon Core. Instrumentation provides realtime metrics of your backend infrastructure whilst the frontend is being measured. This end-to-end measurement is obviously preferred but requires software development work to set up.

