# Overview

The Halo CMM system for Reach and Frequency measurement is an open source implementation of
the [WFA CMM blueprint (2020)](README.md)
.

The open source components of the Halo CMM system are designed to support the Setup Phase and Measurement Phase of the
system.

# Components Overview

## Core components

These components are expected to be used as released by the Halo team, and deployed on every instance of Halo CMM.

<table>
    <tr>
        <td><strong>Component</strong>
        </td>
        <td><strong>Description</strong>
        </td>
    </tr>
    <tr>
        <td>Kingdom - <em>aka Measurement Coordinator</em>
        </td>
        <td>Used to orchestrate measurement requests during the Measurement Phase and coordinate panel data 
             exchange jobs during the Setup Phase.
            <p>
                Used by entities providing the Halo service in a market.
        </td>
    </tr>
    <tr>
        <td>Duchy -<em> aka MPC Node/Aggregator</em>
        </td>
        <td>MPC nodes that make up the MPC consortium for r/f measurement.
            <p>
                Used by MPC operators of a particular instance.
        </td>
    </tr>
    <tr>
        <td>VID Labeler
        </td>
        <td>Library that can be imported by EDPs to label events given a trained VID model
        </td>
    </tr>
    <tr>
        <td>Panellist Exchange Client
        </td>
        <td>Libraries used by the MP and EDP pairs to execute a private double-blind panelist exchange
        </td>
    </tr>

</table>

## Optional Components

These components can be optionally to help entities integrate with core components of Halo CMM. These components can be
modified by integrating entities to suit their needs or used as references to how to integrate with Halo CMM core
components.

<table>
    <tr>
        <td>Privacy Budget Management Library
        </td>
        <td>Reference library and database hooks used by EDPs to maintain Privacy Budgets for Measurement Consumers
        </td>
    </tr>
     <tr>
        <td>FE Reporting/Planning Server
        </td>
        <td>Server that provides services and features to handle scheduling,
            privacy budget optimization and caching of multiple measurement requests & responses.
            <p>
                Used by end user measurement providing entities.
        </td>
    </tr>
    <tr>
        <td>VID training toolkit
        </td>
        <td>Libraries that can be used to help train and VID models.
            <p>
                Used by VID model operators.
        </td>
    </tr>
    <tr>
        <td>Measurement/Reporting CLI(s)</td>
        <td>A set of CLI tools to interact with the measurement coordinator / reporting server.
            <p>Used by administrators and end user measurement providing entities.
        </td>
    </tr>
</table>

# Setup Phase

```mermaid
graph TB;
  subgraph g [ ];
    classDef core fill:#f96;
    classDef optional fill:#96f;
    
      subgraph edp [Event Data Provider];
        direction TB
        edp_model[EDP's VID Model];
        edp_panel_match[Panel Match Client];
        events[(User Events)];
        labeled_events[(Labeled Events)];
        labeler[VID Labeler];
        profiles[(User Profiles)];
        
        class edp_panel_match core;
        class labeler core;
        
        edp_model-->labeler
        events-->labeler
        labeled_events --> edp_panel_match;
        labeler-->labeled_events
        profiles-->labeler
      end
      
      subgraph panel [Panel Operator];
          matched_events[(Matched<p>Panelist Events)];
          panel_keys[(Consented Panelist<p> Match Keys)];
          panel_panel_match[Panel Match Client];

          class panel_panel_match core;
          
          panel_keys-->panel_panel_match;
          panel_panel_match-->matched_events;
      end

      subgraph vid_operator [VID Operator];
        vid_model[VID model];
        vid_training_events[(VID Training Dataset)];
        vid_training_toolkit[VID Training Toolkit];

        class vid_training_toolkit optional;
        
        vid_training_events --> vid_training_toolkit;
        vid_training_toolkit --> vid_model;
      end

      kingdom[<b>Measurement Coordinator</b><p><i>exchange coordinator<i/>];
      panel_shared_storage[(Panel Exchange<p>Private Storage)];

      class kingdom core;

      edp_panel_match --> panel_shared_storage;
      kingdom --> edp_panel_match;
      kingdom --> panel_panel_match;
      kingdom-->edp_model
      matched_events --> vid_training_events;
      panel_panel_match --> panel_shared_storage;
      panel_shared_storage --> edp_panel_match;
      panel_shared_storage --> panel_panel_match;
      vid_model --> kingdom;
  end
```

```mermaid
  graph TB;
      classDef core fill:#f96;
      classDef optional fill:#96f;
      core[Core]
      optional[Optional]
      not_by_halo[Not Built By Halo]
      class core core;
      class optional optional;
```

# Measurement Phase

```mermaid
graph TB;
  classDef core fill:#f96;
  classDef optional fill:#96f;
  subgraph g [ ];
    subgraph edp [Event Data Provider];
      edp_model[EDP's VID Model];
      event_group_uploader[Event Group Uploader];
      events[(User Events)];
      labeled_events[(Labeled Events)];
      labeler[VID Labeler];
      pbm[Privacy Budget Manager];
      profiles[(User Profiles)];
      requisition_fulfiller[Requisition Fulfiller];
      
      class labeler core;
      class pbm optional;
      
      edp_model-->labeler
      events-->labeler
      labeled_events-->requisition_fulfiller;
      labeler-->labeled_events
      pbm --> requisition_fulfiller;
      profiles-->labeler
      requisition_fulfiller --> pbm;
    end

    kingdom[Measurement Coordinator];
    class kingdom core;

    subgraph MPC [MPC Network];
      direction TB;
      aggregator[Aggregator]
      worker1[Worker #1]
      worker2[Worker #2]
      
      class worker1 core;
      class worker2 core;
      class aggregator core;
  
      aggregator-->worker;
      worker1-->worker2
      worker2-->aggregator
    end
       
    rps[Report/Planning server]
    rps_fe[Measurement Client]

    class rps optional

    aggregator-->kingdom;
    event_group_uploader-->kingdom;
    kingdom-->edp_model
    kingdom-->requisition_fulfiller;
    kingdom-->rps;
    kingdom-->worker1;
    rps --> rps_fe;
    rps-->kingdom;
    rps_fe --> rps;
  end
```

```mermaid
  graph TB;
      classDef core fill:#f96;
      classDef optional fill:#96f;
      core[Core]
      optional[Optional]
      not_by_halo[Not Built By Halo]
      class core core;
      class optional optional;
```
