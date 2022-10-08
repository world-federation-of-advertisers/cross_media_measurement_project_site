---
mermaid: true
---

# Overview

The Halo CMM system for Reach and Frequency measurement is an open source implementation of
the [WFA CMM blueprint (2020)](blueprint.html)
.

The open source components of the Halo CMM system are designed to support the Setup Phase and Measurement Phase of the
system.

# Components Overview

## Core components

These components are expected to be used as released by the Halo team, and deployed on every instance of Halo CMM.

<table>
    <thead>
        <td><strong>Component</strong>
        </td>
        <td><strong>Description</strong>
        </td>
    </thead>
    <tr>
        <td>Kingdom - <em>aka Measurement Coordinator</em>
        </td>
        <td>Used to orchestrate measurement requests during the Measurement Phase and coordinate panel data 
             exchange jobs during the Setup Phase.
            <p>
                Used by entities providing the Halo service in a market.
            </p>
        </td>
    </tr>
    <tr>
        <td>Duchy -<em> aka MPC Node/Aggregator</em>
        </td>
        <td>MPC nodes that make up the MPC consortium for r/f measurement.
            <p>
                Used by MPC operators of a particular instance.
            </p>
        </td>
    </tr>
    <tr>
        <td>VID Labeler
        </td>
        <td>Library that can be imported by EDPs to label events given a trained VID model
        </td>
    </tr>
    <tr>
        <td>Panellist Match Client
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
    <thead>
        <td><strong>Component</strong>
        </td>
        <td><strong>Description</strong>
        </td>
    </thead>
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
            </p>
        </td>
    </tr>
    <tr>
        <td>VID training toolkit
        </td>
        <td>Libraries that can be used to help train and VID models.
            <p>
                Used by VID model operators.
            </p>
        </td>
    </tr>
    <tr>
        <td>Measurement/Reporting CLI(s)</td>
        <td>A set of CLI tools to interact with the measurement coordinator / reporting server.
            <p>Used by administrators and end user measurement providing entities.</p>
        </td>
    </tr>
</table>

# Setup Phase

[![](https://mermaid.ink/img/pako:eNqNVU1v2zAM_SuCd1gLtOitQNMghyU5DFuwAO12sYdAsehEiCwZkhysKPrfR33Y8VeG3SzyUSQfH633JFcMklly0LQ6ktcvz5kkxNT7cD6QlPz2JkJyQY1ZQUFypYEUXIjZp-LpcehVleVKUhERT49FRLSXAqtIuj6DtGRFLSVbrc6cgW4SEcK4htxdgwU1NozalVirSNer7WdDfn1dkY07X8IcpKISxK6kNj-mW_dNNu6bLAXHhB2sy2_Sm58GNPHFmNuLV9A9CGC7BvU9nK8BdeqqCaBOG5VWSAI0Wbbx2An3rA3r9gQPIDFP39VScn-_iIB-e2N7vzGCgGH2YWvtHU3QsLthEpAsk-2w_c0kjuJHBZpa5SnyyToUb8J5Xi08lhvbZTsUeII3RC6VNOjAcTRIDIpj_oaITsD_qiGQPIrp032pAVseYQewjgfR_WYR2icJ5b9TkRvitdRlynmD8p2nbDTvzFZTLrk8tCw6xGs0-u0yYG9HaKuUOHGb9tCvwdhbdzIV1q54RE4U4oU1FTsV0lzbxPgOG45OiGCqTOf7xQaoqTWU7sexVEozLh1F84f9AgUw5wv4kx-pPACO7eLlD4uLHsyRahyDQQc9QHoTNLGOcU57mp-pBfISEJ66wEQspJHEcGdd8VM5ntsWrixb1zslquhHd7vuw-UZs93qbCzpfxQ6Zb5S9VXoVAvtUD0iNhQGnNwlJeiScoZv0LsTR5bYI844S2b4yaCgtbBZkskPhNLaqpc3mSczq2u4S-qK4bBWnOIalcmsoMKgFRjHgjbhXfPP28df8PpwAA)](https://mermaid.live/edit#pako:eNqNVU1v2zAM_SuCd1gLtOitQNMghyU5DFuwAO12sYdAsehEiCwZkhysKPrfR33Y8VeG3SzyUSQfH633JFcMklly0LQ6ktcvz5kkxNT7cD6QlPz2JkJyQY1ZQUFypYEUXIjZp-LpcehVleVKUhERT49FRLSXAqtIuj6DtGRFLSVbrc6cgW4SEcK4htxdgwU1NozalVirSNer7WdDfn1dkY07X8IcpKISxK6kNj-mW_dNNu6bLAXHhB2sy2_Sm58GNPHFmNuLV9A9CGC7BvU9nK8BdeqqCaBOG5VWSAI0Wbbx2An3rA3r9gQPIDFP39VScn-_iIB-e2N7vzGCgGH2YWvtHU3QsLthEpAsk-2w_c0kjuJHBZpa5SnyyToUb8J5Xi08lhvbZTsUeII3RC6VNOjAcTRIDIpj_oaITsD_qiGQPIrp032pAVseYQewjgfR_WYR2icJ5b9TkRvitdRlynmD8p2nbDTvzFZTLrk8tCw6xGs0-u0yYG9HaKuUOHGb9tCvwdhbdzIV1q54RE4U4oU1FTsV0lzbxPgOG45OiGCqTOf7xQaoqTWU7sexVEozLh1F84f9AgUw5wv4kx-pPACO7eLlD4uLHsyRahyDQQc9QHoTNLGOcU57mp-pBfISEJ66wEQspJHEcGdd8VM5ntsWrixb1zslquhHd7vuw-UZs93qbCzpfxQ6Zb5S9VXoVAvtUD0iNhQGnNwlJeiScoZv0LsTR5bYI844S2b4yaCgtbBZkskPhNLaqpc3mSczq2u4S-qK4bBWnOIalcmsoMKgFRjHgjbhXfPP28df8PpwAA)
[![](https://mermaid.ink/img/pako:eNplULtuwzAM_BWBXT0HiLqlGbq0HdKpVhCwFlULkEVDpgYjyL-XebhBUA2Hw90RpO4IHXsCC8b8FBx787l5dtlcXpdwmrYUTMeFTIgp2aewXv33eZTIGdMts16Fe0ZH2xeF_aIs4fbjRv6czHL4ng89Jm7fWcymxqQ4m1dV9g9bryed4fGa-ykLUd9laGCgMmD0-tPjecKB9DSQA6vUU8CaxIHLJ41iFd7NuQMrpVIDdfQotI2oDQ1gA6ZJVfJRuLxd27uU2MCI-Yt5yZx-AfJvclY)](https://mermaid.live/edit#pako:eNplULtuwzAM_BWBXT0HiLqlGbq0HdKpVhCwFlULkEVDpgYjyL-XebhBUA2Hw90RpO4IHXsCC8b8FBx787l5dtlcXpdwmrYUTMeFTIgp2aewXv33eZTIGdMts16Fe0ZH2xeF_aIs4fbjRv6czHL4ng89Jm7fWcymxqQ4m1dV9g9bryed4fGa-ykLUd9laGCgMmD0-tPjecKB9DSQA6vUU8CaxIHLJ41iFd7NuQMrpVIDdfQotI2oDQ1gA6ZJVfJRuLxd27uU2MCI-Yt5yZx-AfJvclY)

# Measurement Phase
[![](https://mermaid.ink/img/pako:eNp9VV1r2zAU_StCfdgGKaN5KDSBwdp0Y7CMsK0bzAlGsa5dUVnyJDmjlP73SrLk7zUPsXXPuVfH517ZTziTFPAKF4pU9-jn9XovEMo40XoDOcqkApQzzldn-dXlEJOVYVIQHvCry9zjuj42tQqUoIMP9YJAK5TcnkAYtCGGoJ2SJ0ZBRSJyjLS0mnhyu9m90ejXlw3aunWP4vLTQsm6SuuKS2ILhKKfXRDdheAoRSdv7zQo5Kn6XYdycgQONI2sr836f0SVOFENqbdHdSyTnWInkj2i65oWYNCWCFIMOEpauyAq2YVlbwsFf2ummfM2zWvuzLUbfu-i6FOMdknx6psTVfrurYeQ1dg2bpLdWn9-_iHUGPo3jQ-ds_is_LF_bZ2Y2LMQWRC9XiWaOJUzm-cr2sohHwTdi-b2gYmCyjLZAtG1gtJN0I2UijJBjGz9bawL5OjqaLC3uxuUuL9vYP5J9dD1hjIFmW9cOF3uR4pCQeE2ST62t4eIugKgLpLf_orOLkbIskWWh_kRCBXmRiCUmIM6VR06VWztbGqsR3pbYDmSa4EuvdcF1JeuKm3HvJLKvN9xIoS1G9kzcoLoiyWkOQybxZm9HGI3mqewvHbGIzJQHzoZ52HmbTIhhZWNt4dkDLw2sz1WpSexYN-6fczmDPjH7YITTQ0hcn3Qu4oXuARVEkbte_3J9xCbe2vYHq_sLYWc1Nzs8V48WyqpjfzxKDK8MqqGBa4rSgxsGLFzXeJVTri2UaDMerdtvhX-k7HAFRF_pIyc5xfnZRUb)](https://mermaid.live/edit#pako:eNp9VV1r2zAU_StCfdgGKaN5KDSBwdp0Y7CMsK0bzAlGsa5dUVnyJDmjlP73SrLk7zUPsXXPuVfH517ZTziTFPAKF4pU9-jn9XovEMo40XoDOcqkApQzzldn-dXlEJOVYVIQHvCry9zjuj42tQqUoIMP9YJAK5TcnkAYtCGGoJ2SJ0ZBRSJyjLS0mnhyu9m90ejXlw3aunWP4vLTQsm6SuuKS2ILhKKfXRDdheAoRSdv7zQo5Kn6XYdycgQONI2sr836f0SVOFENqbdHdSyTnWInkj2i65oWYNCWCFIMOEpauyAq2YVlbwsFf2ummfM2zWvuzLUbfu-i6FOMdknx6psTVfrurYeQ1dg2bpLdWn9-_iHUGPo3jQ-ds_is_LF_bZ2Y2LMQWRC9XiWaOJUzm-cr2sohHwTdi-b2gYmCyjLZAtG1gtJN0I2UijJBjGz9bawL5OjqaLC3uxuUuL9vYP5J9dD1hjIFmW9cOF3uR4pCQeE2ST62t4eIugKgLpLf_orOLkbIskWWh_kRCBXmRiCUmIM6VR06VWztbGqsR3pbYDmSa4EuvdcF1JeuKm3HvJLKvN9xIoS1G9kzcoLoiyWkOQybxZm9HGI3mqewvHbGIzJQHzoZ52HmbTIhhZWNt4dkDLw2sz1WpSexYN-6fczmDPjH7YITTQ0hcn3Qu4oXuARVEkbte_3J9xCbe2vYHq_sLYWc1Nzs8V48WyqpjfzxKDK8MqqGBa4rSgxsGLFzXeJVTri2UaDMerdtvhX-k7HAFRF_pIyc5xfnZRUb)
[![](https://mermaid.ink/img/pako:eNplULtuwzAM_BWBXT0HiLqlGbq0HdKpVhCwFlULkEVDpgYjyL-XebhBUA2Hw90RpO4IHXsCC8b8FBx787l5dtlcXpdwmrYUTMeFTIgp2aewXv33eZTIGdMts16Fe0ZH2xeF_aIs4fbjRv6czHL4ng89Jm7fWcymxqQ4m1dV9g9bryed4fGa-ykLUd9laGCgMmD0-tPjecKB9DSQA6vUU8CaxIHLJ41iFd7NuQMrpVIDdfQotI2oDQ1gA6ZJVfJRuLxd27uU2MCI-Yt5yZx-AfJvclY)](https://mermaid.live/edit#pako:eNplULtuwzAM_BWBXT0HiLqlGbq0HdKpVhCwFlULkEVDpgYjyL-XebhBUA2Hw90RpO4IHXsCC8b8FBx787l5dtlcXpdwmrYUTMeFTIgp2aewXv33eZTIGdMts16Fe0ZH2xeF_aIs4fbjRv6czHL4ng89Jm7fWcymxqQ4m1dV9g9bryed4fGa-ykLUd9laGCgMmD0-tPjecKB9DSQA6vUU8CaxIHLJ41iFd7NuQMrpVIDdfQotI2oDQ1gA6ZJVfJRuLxd27uU2MCI-Yt5yZx-AfJvclY)
