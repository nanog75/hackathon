---
layout: homepage
permalink: /
author_profile: true
author: "Nanog75 Hackathon Team"
sitemap: true
date: null

---

{% include base_path %}

{% include base_path %}


<div class="feature__wrapper" style="margin-bottom: 100px;">
    <div class="feature__item--center">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="height: 500px; width: 1000px; display: block; margin-left: auto; margin-right: auto;">
            <img src="{{ base_path }}/images/topology_nanog75_with_logos.png" alt="" />
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title">Network Topology for today</h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p style="text-align: center;">The topology consists of 4 Cisco IOS-XR routers in a diamond topology and 3 development instances, each meant for a different stage of the deployment cycle. All the instances per pod are running on the Tesuto platform in the cloud.</p>
            </div>
            <p><a target="_blank" href="{{ base_path }}/assets/NANOG75_Hackathon_Lab_Info.pdf" class="btn btn--large">Connect to your Pod!</a></p>
        </div>
      </div>
    </div>
</div>

<hr/>

<div class="feature__wrapper">
    <div class="feature__item--center">
      <div class="archive__item">
          <div class="archive__item-teaser center" style="height: 600px; width: 1000px; display: block; margin-left: auto; margin-right: auto;">
            <img src="{{ base_path }}/images/team_groups.png" alt="" />
          </div>
        <div class="archive__item-body">
            <h2 class="archive__item-title">Splitting up but working together!</h2>
            <div class="archive__item-excerpt" style="font-size: 0.65em;">
              <p style="text-align: left;"><b>Each team will divide up into 3 groups</b> focused on different parts of the Network Deployment cycle. Work separately with an aim to come together at the end. By the end your team should be able to fully bootstrap all the routers using Yang models, automate provisioning using a single Ansible playbook and remediate LSP paths based on selected events in the network - without touching the CLI of the routers<ul><li><b>Day0 Group</b> will take the ZTP node in the topology and focus on bootstrapping all the routers (starting with rtr2) in the topology to a pre-defined configuration using only Yang models in a ZTP script</li><li><b>Day1 Group</b> will focus on node dev1 and work with Ansible, YDK, GNMI and Open/R to bring up BGP sessions and set up telemetry data for Day2 group to parse</li><li><b>Day3 Group</b> will focus on integrating Telemetry data and application events into their controller application that leverages gRIBI to control LSP paths in the network.</li></ul></p>
            </div>
            <p>
            <a target="_blank" href="{{ base_path }}/day0-instructions" class="btn btn--large"> Day0</a>
            <a target="_blank" href="{{ base_path }}/day1-instructions" class="btn btn--large"> Day1</a>
            <a target="_blank" href="{{ base_path }}/day2-instructions" class="btn btn--large"> Day2</a></p>
        </div>
      </div>
    </div>
</div>

