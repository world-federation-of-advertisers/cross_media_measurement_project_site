### _[Draft For WFA | Review In Progress]_

# Private Reach & Frequency Estimators Evaluation Results

<br>
<table align="center">
  <tr>
   <td align="center"><strong>Craig Wright</strong>
   </td>
   <td align="center"><strong>Aaron An</strong>
   </td>
   <td align="center"><strong>Xichen Huang</strong>
   </td>
  </tr>
  <tr>
  </tr>
  <tr>
   <td align="center">Google, LLC
   </td>
   <td align="center">Facebook Inc.
   </td>
   <td align="center">Google, LLC
   </td>
  </tr>
</table>

<br>

<center>
  Additional contributors to evaluation code base and design in Appendix G
</center>


<br>

## Table of Contents
- [**Executive Summary**](#executive-summary)
- [**Introduction**](#introduction)
- [**Candidates and Evaluation Methods**](#candidates-and-evaluation-methods)
    - [Sketch and Estimator Candidates](#sketch-and-estimator-candidates)
        - [Bloom Filter Variants](#bloom-filter-variants)
        - [Non-Bloom-filter-based Private Sketches](#non-bloom-filter-based-private-sketches)
    - [Theme, Scenario and Setting](#theme-scenario-and-setting)
- [**Evaluation Results Summary**](#evaluation-results-summary)
    - [No privacy theme](#no-privacy-theme)
        - [Reach Estimation Under No Privacy Theme](#reach-estimation-under-no-privacy-theme)
        - [Frequency Estimation Under No Privacy Theme](#frequency-estimation-under-no-privacy-theme)
    - [Local privacy theme](#local-privacy-theme)
        - [Reach Estimation Under Local Privacy Theme](#reach-estimation-under-local-privacy-theme)
        - [Frequency Estimation Under Local Privacy Theme](#frequency-estimation-under-local-privacy-theme)
    - [Global privacy theme](#global-privacy-theme)
        - [Reach Estimation Under Global Privacy Theme](#reach-estimation-under-global-privacy-theme)
        - [Frequency Estimation Under Global Privacy Theme](#frequency-estimation-under-global-privacy-theme)
- [**Conclusions and Next Steps**](#conclusions-and-next-steps)
- [**References**](#references)
- [**End Notes**](#end-notes)
- [**Appendix**](#appendix)
    - [Appendix A: Detailed Parameterization of the Sketch and Estimator Candidates](#appendix-a-detailed-parameterization-of-the-sketch-and-estimator-candidates)
    - [Appendix B: Example plots showing VoC has bias when the sets have non-homogeneous correlation:](#appendix-b-example-plots-showing-voc-has-bias-when-the-sets-have-non-homogeneous-correlation)
    - [Appendix C: ADBF sketch intersection operation bias](#appendix-c-adbf-sketch-intersection-operation-bias)
    - [Appendix D: Stratified VoC with clipping vs without clipping](#appendix-d-stratified-voc-with-clipping-vs-without-clipping)
    - [Appendix E: Boxplots of number of sets vs relative error.](#appendix-e-boxplots-of-number-of-sets-vs-relative-error)
    - [Appendix F: Bloom filters sensitive to input set size](#appendix-f-bloom-filters-sensitive-to-input-set-size)
    - [Appendix G: Some Thoughts on Privacy Budgets](#appendix-g-some-thoughts-on-privacy-budgets)
    - [Appendix H: Contributors](#appendix-h-contributors)

<br>

## Executive Summary

This paper is the culmination of an extensive evaluation of privacy-preserving cross-media reach and frequency estimation algorithms aimed at deduplicating unique reach and frequency across 20 or more channels. The work presented herein was conducted at the behest of the World Federation of Advertisers (WFA) and in conjunction with the Media Ratings Council (MRC) from April to November 2020, and will continue with a peer review session in order to gather feedback and further discuss the evaluation results with industry peers.

As discussed in [the WFA’s Cross-Media Measurement System for Reach and Frequency](https://github.com/world-federation-of-advertisers/cardinality_estimation_evaluation_framework/blob/master/doc/cardinality_and_frequency_estimation_evaluation_framework.md) [[1]](#references), all reach and frequency estimators considered by this evaluation, referred to as candidates, are sketch-based algorithms, a sketch being a compact data structure for representing a multi-set of aggregated cross-media measurement data. Specifically we consider differentially private sketches, which are sketches that can be constructed to provide a mathematical privacy guarantee.

In order to provide such privacy guarantees, these privacy preserving sketching methods necessarily require the injection of noise into the measurement process, however there is a choice as to where in the measurement process noise is injected. We refer to this choice as a privacy theme, which considers whether noise is added when the sketches are created by data providers (local privacy theme) or whether noise is added when the sketches are pooled and combined (global privacy theme).

The first choice allows sketches to be released immediately upon creation and combined as desired with other sketches, whereas the second choice requires that sketches be combined via a secure multiparty computation (MPC) protocol. Such cryptographic protocols can be constructed to add the requisite noise to the output measurement while calculating it, but unfortunately, construction of an efficient protocol is not possible for all types of sketches (which impacted which sketches were tested within each theme). This evaluation aims to find an optimal combination of privacy theme and private sketch algorithm for estimating cross-media reach and frequency.

Overall this evaluation has determined that locally private sketches have serious drawbacks when it comes to accuracy. Our best locally private candidates are biased when inputs are correlated, while the others are sensitive to the size of the sets being measured. All methods exhibit increased error rates as the number of inputs increases. This combined lack of robustness and inability to scale makes these methods unsuitable for supporting a general purpose system for cross-media reach and frequency estimation.

On the other hand, some sketching methods within the globally private theme are remarkably accurate. In particular the [LiquidLegions sketch](https://research.google/pubs/pub49177/) [[2]](#references), which is based upon an exponentially distributed counting Bloom filter, allows for privacy-preserving reach and frequency estimation that is highly accurate, while being invariant to the number of inputs, the distribution of the inputs, and the maximum frequency being measured. Unfortunately, as compared to the locally private methods, this accuracy comes with substantially higher computational costs, a full understanding of which will require an end-to-end synthetic data test at scale, however experience with similar systems and early performance testing done in private give reason to believe that the operational costs will be reasonable. Moreover, an open source reference implementation of the LiquidLegions MPC protocol will be available soon, at which time these initial test results will also be made available and replicated as necessary. Once this is done we recommend proceeding with an end-to-end synthetic data test of the LiquidLegions sketch and its MPC protocol at scale.

<br>

## Introduction

The estimation of cross-media reach and frequency is the process of finding the number of unique users reached by an advertisement on different media and the number of times those exposures happened. This functionality is enabled by the Private Reach and Frequency Estimator (PRFE), which is an algorithm that comprises two major components: 1) a private sketch that is an aggregate representation of the reached audience; 2) an estimator that helps find the reach and frequency of combined private sketches from different media. The Private Reach and Frequency Estimator is one of the four primary components outlined in the WFA's recently published Cross-Media Measurement System tech blueprint. It is downstream of the Virtual People ID model, which is responsible for synchronizing the identity foundation of the reached audiences from different data providers. It then generates private sketches using the Virtual People IDs and sends these aggregate representations downstream where the reach and frequency estimates are made available.

<br>

<p align="center">
<img src="image1.png?raw=true">
</p>

<br>


The goal of this Private Reach and Frequency Estimator evaluation is to find an algorithm that enables accurate measurement while minimizing the risk of user re-identification. In our previous papers, we introduced a family of candidates methods for the reach and frequency estimation in detail [[1]](#references). We also introduced an evaluation framework [[2]](#references) that assesses the accuracy, scalability and privacy protection of different candidate methods in realistic cross-media campaign scenarios. In this paper, we present the results of that evaluation effort and provide a recommendation for private sketches and algorithms for cross-media measurement.

It is a complex process to determine which private cardinality and frequency estimator is optimal for cross-media measurement. A central trade-off featured in the evaluation process is between the estimator’s utility (accuracy of the estimation) and privacy guarantees. The employment of privacy protection mechanisms, adding differential privacy noise, for instance, often hurts the accuracy of the reach and frequency estimator. Different reach and frequency estimators are impacted by this tradeoff at different scales. An ideal algorithm should be accurate in estimating reach and frequency even with strong privacy guarantees. In the meantime it should be scalable where its merits should not vary in different cross-media scenarios and without incurring heavy computational cost.

The rest of this paper is organized as follows. First we review the various candidates for Reach and Frequency estimation and the evaluation methods. Then we summarize the set of privacy themes, scenarios and settings of evaluation, which provide an evaluation structure that comprehensively mimics realistic cross-media campaigns. Next we report the evaluation results for the various candidates across privacy themes and cross-media campaign scenarios. Finally, we provide an overall recommendation for the privacy preserving reach and frequency estimator.

<br>

## Candidates and Evaluation Methods

<br>

### Sketch and Estimator Candidates

Several candidate reach and frequency estimators were considered for cross-media measurement. Before we report the results of the evaluation, it is helpful to first introduce the theories and characteristics of these estimators for better understanding of the results. These candidates can be categorized into two groups: Bloom filter variants and non-Bloom-filter-based private sketches.

#### Bloom Filter Variants

The first group of estimators are different variants of Bloom filter, which is a well-known algorithm for cardinality estimation. Standard Bloom filters are uniformly distributed and provide good accuracy in estimating cardinality. However, the size of a uniform Bloom filter grows linearly with the cardinality of the set that needs to be measured, and therefore poses a scalability problem. To overcome this we developed a set of methods for applying arbitrary distributions to the register allocation for Bloom filters. For reach estimation, a few major candidates include: Exponential Bloom Filters, Logarithmic Bloom Filters and Geometric Bloom filters, which are named according to the distribution that the Bloom filter registers follow. More details about the estimation methods can be found in [[1]](#references).

To account for frequency, a counting Bloom filter (CBF) is used. Unfortunately CBFs with non-uniform distributions are susceptible to a large number of collisions where the probability density is highest [<sup>1</sup>](#end-notes). This in turn skews the frequency distribution to the right. For this reason we developed a technique that we call the same key aggregator. Essentially this technique allows the CBF to keep track of the fingerprint of the item that hashes to each bucket of the CBF and to invalidate the count when there is a collision. Notably, this technique can be implemented in the clear or as part of a cryptographic protocol, and it is capable of estimating any distribution, not just the frequency distribution, even multiple distributions simultaneously. An exponentially distributed counting Bloom filter that uses the same key aggregator is referred to as _LiquidLegions_, and while this technique was designed for frequency estimation, it can be used in the same fashion as a regular exponential Bloom filter to estimate reach as well. See [[2]](#references) for additional details.

#### Non-Bloom-filter-based Private Sketches

The second group of estimators are private sketches that are not based on the Bloom filter. These include the following: Conditional Independence Model, Vector of Counts, Meta-VoC, HLL++, Stratified sketch.

**The Conditional Independence Model** (Cond. Ind.) simply looks at the set sizes across publishers and assumes independence to arrive at an estimate. One issue with it is that it requires the universe size as input, which was provided in simulation. However, in practice the universe size may not be available.

**Vector of Counts** (VoC) is another type of sketch that summarizes the reached users as a set of bucketed counts. By choosing an appropriate granularity, i.e., number of buckets, k-anonymity is guaranteed, and meanwhile, the correlation between publishers is well preserved. Differential privacy can be achieved by adding noise to the count of each bucket. VoC deduplicates the reach between any two publishers without bias, and deduplicates more publishers using a model-based, scalable approach called Sequential VoC. In contrast to the conditional independence model, Sequential VoC is a parameter-free model and does not depend upon knowing the universe size.

**Meta-VoC** is a technique that creates Vector of Counts sketches from a Bloom filter by inserting the Bloom filter register IDs into a VoC sketch. Then the VoC is used to estimate the number of active Bloom filter registers. This count can then be used with the Bloom filter estimator to estimate cardinality. This method was developed as a way to parallelize the construction of Bloom filters and because we think it is possible to generate such VoC sketches from a secure multiparty computation protocol whose primary inputs are Bloom filters.

**Hyper Log Log ++** (HLL++) is characterized by low error rates and low memory consumption making it the de facto standard for estimating cardinality. Unfortunately, despite being an active area of research, a privacy preserving version of HLL++ that would meet the requirements of the cross media measurement system has not been developed. We have chosen to include it here as a baseline due to its widespread use and excellent performance.

The final method considered is really a meta-technique called **stratified sketch**. Stratified sketches are composed of multiple atomic sketches, one per frequency. The sketches that compose it can be of any type as long as they support a union, intersection, and difference operation. This technique was developed specifically for frequency estimation where the underlying sketch could not support the estimation of frequency directly.

For the detailed parameterizations that were used for each estimator, please refer to the table in Appendix A. The abbreviations listed in that table are used in the results tables in subsequent sections.

<br>

### Theme, Scenario and Setting

To assess the performance of the different estimators we developed a structure that evaluates them across a combination of different themes, scenarios and settings. Proposed in [the evaluation framework](https://github.com/world-federation-of-advertisers/cardinality_estimation_evaluation_framework/blob/master/doc/cardinality_and_frequency_estimation_evaluation_framework.md) design, we considered three privacy themes to guide the evaluation of the candidate estimators. The first theme (no-privacy theme) evaluates each candidate without any noise added to the output in order to establish a baseline for accuracy. The second theme (local privacy theme) adds enough noise to the sketches so that they may be seen unencrypted. For this purpose either the Laplace mechanism or Bloom-and-Flip are used. The third theme (global privacy theme) assumes sketches from publishers will be pooled and combined through a secure multi-party computation (MPC) protocol, and that noise is added just once for the entire set of sketches. For the global theme, the geometric mechanism is used to generate noise. Here local and global refer to terms from the differential privacy literature, and are akin to individual sketch level and pooled privacy respectively. They have absolutely no geographic connotation.

Across all themes, we designed a set of different scenarios to either represent real-world cross-media campaigns or stress test the estimators. To differentiate use cases within each scenario, we defined a set of parameters to represent different settings for each scenario. Each setting was simulated 100 times and the results for each were written into a detailed transcript. Reach estimators were compared on the basis of average relative error while frequency estimators were compared on the basis of average shuffle distance [<sup>2</sup>](#end-notes). The variance of the results was also captured.

The following table shows a summary of the reach scenarios considered. In all scenarios a universe size of 1 million, with a reach rate of 20% for large sets and 1% for small sets was used. The maximum number of sets unioned was 20. We used this smaller universe size for the sake of computational efficiency, as simulating larger universe sizes would have made generating results for all of our scenarios and sketches infeasible. This was adequate for exploring the accuracy of reach estimates under different input data distributions, and was sufficient to rule out many candidates. For our best candidate, which has performance independent of its input data distribution, we expanded the audience size up to 10 million and found it to perform well. Due to its distribution-agnostic performance property, we believe it will also perform well with larger size input (eg. at 100M magnitude), which we will evaluate further in future tests.

The noise parameter for differential privacy, called epsilon, can be varied from 0 to infinity, where 0 is essentially all noise and infinity is noiseless. When considering the value of epsilon we focused on the highest value that would be acceptable to the parties doing the evaluation were these methods to be put into production. That value is ln(3), or about 1.09. A nice property of this value is that when applied to Bloom filters it results in a 0.25 probability of flipping the value of each bucket from a 0 to a 1 or a 1 to a 0.

<br>

<table>
  <tr>
   <td><strong>Scenario</strong>
   </td>
   <td><strong>Index</strong>
   </td>
   <td><strong>Number of settings</strong>
   </td>
  </tr>
  <tr>
   <td>independent
   </td>
   <td>1
   </td>
   <td>6
   </td>
  </tr>
  <tr>
   <td>remarketing
   </td>
   <td>2
   </td>
   <td>6
   </td>
  </tr>
  <tr>
   <td>exponential_bow-user_activity_association:independent
   </td>
   <td>3a
   </td>
   <td>6
   </td>
  </tr>
  <tr>
   <td>exponential_bow-user_activity_association:identical
   </td>
   <td>3b
   </td>
   <td>6
   </td>
  </tr>
  <tr>
   <td>fully_overlapped
   </td>
   <td>4a
   </td>
   <td>2
   </td>
  </tr>
  <tr>
   <td>subset
   </td>
   <td>4b
   </td>
   <td>3
   </td>
  </tr>
  <tr>
   <td>sequentially_correlated-order:random-correlated_sets:one
   </td>
   <td>5a
   </td>
   <td>36
   </td>
  </tr>
  <tr>
   <td>sequentially_correlated-order:random-correlated_sets:all
   </td>
   <td>5b
   </td>
   <td>36
   </td>
  </tr>
</table>

<br>

For each reach scenario we report a box plot of relative error across 100 runs. We then identify the maximum number of publishers that a candidate estimator can support for a desired performance. For example, the 95/5 performance criteria requires relative errors less than 5% for 95 of the 100 random runs. Finally, we build a per scenario summary result by averaging the maximum number of publishers over parameter sets. The average maximum number of publishers indicates how many publishers a candidate estimator can support for a given performance level.

For each frequency scenario, we report a box plot of shuffle distance across 100 runs. We then identify the maximum number of publishers that a candidate estimator can support for a particular max frequency and the desired performance level. We considered two values for max frequency, 5 and 15. The following table shows a summary of the frequency scenarios considered.

<br>

<table>
  <tr>
   <td><strong>Scenario</strong>
   </td>
   <td><strong>Index</strong>
   </td>
   <td><strong>Number of settings</strong>
   </td>
  </tr>
  <tr>
   <td>homogeneous
   </td>
   <td>1
   </td>
   <td>12
   </td>
  </tr>
  <tr>
   <td>heterogeneous
   </td>
   <td>2
   </td>
   <td>12
   </td>
  </tr>
  <tr>
   <td>publisher_constant_frequency
   </td>
   <td>3
   </td>
   <td>4
   </td>
  </tr>
</table>

<br>

Simulated data was used to represent users' identifiers instead of real cross-media campaign data, and we believe that this is sufficient to determine the performance of the various candidates. It also makes it easier for other researchers to reproduce the results. Moreover, there are theoretical results for many of the candidates, including our best performing one, that show their accuracy to be invariant to the distribution of inputs. Moreover, our simulations are in agreement with the theory. See [[2]](#references) for a detailed explanation of evaluation criteria as well as simulation scenarios and settings.

<br>

## Evaluation Results Summary

This section provides a summary of key results across all three privacy themes for reach and frequency estimation.

<br>

### No privacy theme

Since estimators in this theme do not protect privacy, they are not viable as final recommendations, however they do provide a baseline for understanding the impact of privacy preserving techniques on overall accuracy.

#### Reach Estimation Under No Privacy Theme

Under this theme we evaluated the following reach estimators.


*   HLL++
*   Exponential Bloom filter (LiquidLegions)
*   Log Bloom filter
*   Geometric Bloom filter
*   Conditional independence
*   Vector of Counts

As expected HLL++ provides accurate unbiased estimates under all scenarios. Bloom filter-based techniques also do well under this theme. Regardless of the register distribution, they provide highly accurate estimates (95/5 criteria) across at least 20 publishers, and they show no bias regardless of scenario.

These results are consistent with the theory for both HLLs and Bloom filters, which show them to be invariant to input data distribution (i.e. scenario) and to have accuracy independent of the number of sets being unioned. That is, without differential privacy noise, they are capable of unioning any number of sets without any degradation in accuracy.

The other methods do not perform as well and are impacted by the input data distribution. One of these is the simple conditional independence estimator, which as might be expected, does well when independence actually exists, but fares rather poorly otherwise.

Vector of Counts does much better than the simple conditional independence estimator, but it can exhibit significant bias when input sets are correlated (see Appendix B). Moreover, it is unable to estimate reach across more than 20 publishers at our 95/5 accuracy threshold in all but one scenario.

Meta VoC does reasonably well when constructed using a uniform Bloom filter, but fares poorly when constructed using an exponential Bloom filter. The reason for this is that the VoC merge operation shows bias under correlation and we see a high degree of correlation between the active register sets in exponential Bloom filters. Such correlations do not occur for uniform Bloom filters except where the underlying data set exhibits correlation. Regardless, standard VoC outperforms all flavors of Meta VoC.

The following table provides a summary of the results for this theme. It shows the average number of estimable sets for each estimator across the set of scenarios given the 95/5 quality criteria. Cells that are colored green allow for 18+ sets, cells colored yellow allow for 11-18 sets, and red cells allow for less than 11.

<br>

<table>
  <tr>
   <td>95/5 criteria
   </td>
   <td>Meta VoC Uni
   </td>
   <td>Meta VoC Exp
   </td>
   <td>Exp BF 10
   </td>
   <td>Geo BF
   </td>
   <td>HLL++
   </td>
   <td>Log BF
   </td>
   <td>Cond. Ind.
   </td>
   <td>VoC
   </td>
  </tr>
  <tr>
   <td>1.independent-universe_size:1000000-small_set:10000
   </td>
   <td><p style="text-align: right">
13.83</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
17.67</p>

   </td>
  </tr>
  <tr>
   <td>2.remarketing-remarketing_size:200000-universe_size:1000000
   </td>
   <td><p style="text-align: right">
17.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
1.83</p>

   </td>
   <td><p style="text-align: right">
18.83</p>

   </td>
  </tr>
  <tr>
   <td>3a.exponential_bow-user_activity_association:independent-universe_size:1000000
   </td>
   <td><p style="text-align: right">
14.33</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
19.17</p>

   </td>
  </tr>
  <tr>
   <td>3b.exponential_bow-user_activity_association:identical-universe_size:1000000
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
5.83</p>

   </td>
   <td><p style="text-align: right">
10.33</p>

   </td>
  </tr>
  <tr>
   <td>4a.fully_overlapped-universe_size:1000000-num_sets:20
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>4b.subset-universe_size:1000000-order:random
   </td>
   <td><p style="text-align: right">
8.33</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
8.33</p>

   </td>
  </tr>
  <tr>
   <td>5a.sequentially_correlated-order:random-correlated_sets:one
   </td>
   <td><p style="text-align: right">
10.31</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
0.06</p>

   </td>
   <td><p style="text-align: right">
10.22</p>

   </td>
  </tr>
  <tr>
   <td>5b.sequentially_correlated-order:random-correlated_sets:all
   </td>
   <td><p style="text-align: right">
9.72</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
1.31</p>

   </td>
   <td><p style="text-align: right">
11.83</p>

   </td>
  </tr>
</table>

<br>

#### Frequency Estimation Under No Privacy Theme

For frequency the following estimators were considered. This set varies from the set evaluated above for reach because most of the above sketches do not support frequency estimation.

*   LiquidLegions
*   Stratified sketch of Vector of Counts
*   Stratified sketch of exponential Bloom filters

In the case of frequency we considered three candidates. The first was the LiquidLegions sketch which was able to estimate frequency across 10 publishers for both values of max frequency. As with reach, this is predicated by the theory, which further indicates that there is no upper limit on the number of publishers that can be unioned.

The second method considered was a stratified sketch of Vector of Counts. It allows for at least pairwise estimation of frequency for both values of max frequency, but as with reach, its accuracy is sensitive to the input distribution.

The last method, which was not exhaustively simulated, was a stratified sketch of exponential Bloom filters. We chose exponential Bloom filters because for reach they had results that were as good or better than any other Bloom filter distribution. Unfortunately due to bias in the intersection operation they perform very poorly (see Appendix C). This was determined after just a few simulations and due to these early results we did not see any practical value in generating a full set of results. Cells that are colored green allow for 10+ sets, cells colored yellow allow for 6-9 sets, and red cells allow for less than 6.

<br>

<table>
  <tr>
   <td>95/5 Criteria
   </td>
   <td>LiquidLegions
<p>
(max freq = 15)
   </td>
   <td>LiquidLegions
<p>
(max freq = 5)
   </td>
   <td>Strat VoC Clip
<p>
(max freq = 15)
   </td>
   <td>Strat VoC Clip
<p>
(max freq = 5)
   </td>
  </tr>
  <tr>
   <td>1.homogeneous-universe_size:200000-num_sets:10
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
4.25</p>

   </td>
   <td><p style="text-align: right">
6.08</p>

   </td>
  </tr>
  <tr>
   <td>2.heterogeneous-universe_size:200000-num_sets:10
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
2.67</p>

   </td>
   <td><p style="text-align: right">
3.83</p>

   </td>
  </tr>
  <tr>
   <td>3.publisher_constant_frequency-universe_size:200000-num_sets:10
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
</table>

<br>

### Local privacy theme

Finding an excellent candidate in the local privacy theme has proven elusive. This is especially the case for estimating frequency. The problem comes from adding noise to each sketch, which then compounds in proportion to the number of sketches being unioned. In the case of frequency the problem is exacerbated by the fact that a single user is represented according to their frequency. This means that in the naive case, query sensitivity is proportional to the user with the maximum frequency.[^3]

Stratified sketches get around the query sensitivity problem mentioned above by creating one sketch per frequency per publisher. The consequences of this are an exponential growth in the number of sketch operations that must be undertaken as the frequency to be estimated increases. This is less a computational problem as intermediate results can be cached and reused. Rather, the main problem is that each set operation increases the amount of accumulated noise.

#### Reach Estimation Under Local Privacy Theme

Under the local privacy theme we considered the following reach estimators.

*   Exponential Bloom filter with a decay rate of 2
*   Exponential Bloom filter with a decay rate of 10 (LiquidLegions)
*   Log Bloom filter
*   Geometric Bloom filter
*   Conditional independence
*   Vector of Counts
*   Meta VoC with a Uniform Bloom filter
*   Meta VoC with an Exponential Bloom filter

In this theme we considered two performance criteria 95/5 and 95/10. The reason for this is that at 95/5 the performance of all methods was poor and we simply wanted to understand how they behaved under a relaxed quality regime. Tables for both quality regimes are included.

Bloom filter variants perform similarly across both quality criteria, however we note that the choice of decay rate becomes much more important in the local theme. An important factor that impacts Bloom filter accuracy in this theme is the size of the Bloom filter with respect to the size of the set being measured. That is, given a Bloom filter of constant size, the accuracy after adding noise will be dependent upon the size of the set being estimated. If we recall that only Bloom filters of the same size can be unioned then a serious practical consequence is that unioning noisy Bloom filters whose input sets differ substantially in size is problematic. Specifically, unioning smaller sets to one another will cause the noise to grow faster (see Appendix F). In short, this property violates our desire to have the cross media measurement technology work equally well for impression data providers of all sizes.

Conditional independence works well when there is actual independence and rather poorly otherwise. It is not a serious contender in this theme.

Vector of counts performs very well across many scenarios, but as in the no privacy theme it exhibits significant bias when input data is correlated, for example in the sequentially correlated scenario. Nonetheless, it is the most accurate and scalable candidate for estimating reach under this theme.

The following tables show the performance of the estimators under the 95/5 and 95/10 criteria, with epsilon equal to `ln(3)`.

<br>

<table>
  <tr>
   <td><em>95/5 Criteria</em>
<p>
<em>epsilon=ln(3)</em>
   </td>
   <td>Meta VoC Uni
   </td>
   <td>Meta VoC Exp
   </td>
   <td>Exp BF 10
   </td>
   <td>Exp BF 2
   </td>
   <td>Geo BF
   </td>
   <td>Log BF
   </td>
   <td>Cond Ind.
   </td>
   <td>VoC
   </td>
  </tr>
  <tr>
   <td>1.independent-universe_size:1000000-small_set:10000
   </td>
   <td><p style="text-align: right">
13.17</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.33</p>

   </td>
   <td><p style="text-align: right">
2.83</p>

   </td>
   <td><p style="text-align: right">
2.67</p>

   </td>
   <td><p style="text-align: right">
2.50</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
18.50</p>

   </td>
  </tr>
  <tr>
   <td>2.remarketing-remarketing_size:200000-universe_size:1000000
   </td>
   <td><p style="text-align: right">
14.50</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.33</p>

   </td>
   <td><p style="text-align: right">
3.00</p>

   </td>
   <td><p style="text-align: right">
3.00</p>

   </td>
   <td><p style="text-align: right">
2.33</p>

   </td>
   <td><p style="text-align: right">
1.83</p>

   </td>
   <td><p style="text-align: right">
14.67</p>

   </td>
  </tr>
  <tr>
   <td>3a.exponential_bow-user_activity_association:independent-universe_size:1000000
   </td>
   <td><p style="text-align: right">
13.33</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
2.67</p>

   </td>
   <td><p style="text-align: right">
2.83</p>

   </td>
   <td><p style="text-align: right">
2.33</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
18.00</p>

   </td>
  </tr>
  <tr>
   <td>3b.exponential_bow-user_activity_association:identical-universe_size:1000000
   </td>
   <td><p style="text-align: right">
10.17</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.33</p>

   </td>
   <td><p style="text-align: right">
3.00</p>

   </td>
   <td><p style="text-align: right">
3.33</p>

   </td>
   <td><p style="text-align: right">
2.50</p>

   </td>
   <td><p style="text-align: right">
5.83</p>

   </td>
   <td><p style="text-align: right">
10.33</p>

   </td>
  </tr>
  <tr>
   <td>4a.fully_overlapped-universe_size:1000000-num_sets:20
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
1.00</p>

   </td>
   <td><p style="text-align: right">
3.00</p>

   </td>
   <td><p style="text-align: right">
3.00</p>

   </td>
   <td><p style="text-align: right">
2.50</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
  <tr>
   <td>4b.subset-universe_size:1000000-order:random
   </td>
   <td><p style="text-align: right">
3.33</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.67</p>

   </td>
   <td><p style="text-align: right">
2.00</p>

   </td>
   <td><p style="text-align: right">
2.00</p>

   </td>
   <td><p style="text-align: right">
1.33</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
7.33</p>

   </td>
  </tr>
  <tr>
   <td>5a.sequentially_correlated-order:random-correlated_sets:one
   </td>
   <td><p style="text-align: right">
9.97</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.39</p>

   </td>
   <td><p style="text-align: right">
2.67</p>

   </td>
   <td><p style="text-align: right">
2.97</p>

   </td>
   <td><p style="text-align: right">
2.31</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
10.11</p>

   </td>
  </tr>
  <tr>
   <td>5b.sequentially_correlated-order:random-correlated_sets:all
   </td>
   <td><p style="text-align: right">
9.25</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.44</p>

   </td>
   <td><p style="text-align: right">
3.25</p>

   </td>
   <td><p style="text-align: right">
3.31</p>

   </td>
   <td><p style="text-align: right">
2.39</p>

   </td>
   <td><p style="text-align: right">
1.17</p>

   </td>
   <td><p style="text-align: right">
11.67</p>

   </td>
  </tr>
</table>

<br>


<table>
  <tr>
   <td>95/10 criteria
<p>
epsilon = ln(3)
   </td>
   <td>Meta VoC Uni.
   </td>
   <td>Meta VoC Exp
   </td>
   <td>Exp BF 10
   </td>
   <td>Exp BF 2
   </td>
   <td>Geo BF
   </td>
   <td>Log BF
   </td>
   <td>Cond. Ind.
   </td>
   <td>VoC
   </td>
  </tr>
  <tr>
   <td>1.independent-universe_size:1000000-small_set:10000
   </td>
   <td><p style="text-align: right">
19.67</p>

   </td>
   <td><p style="text-align: right">
1.00</p>

   </td>
   <td><p style="text-align: right">
2.50</p>

   </td>
   <td><p style="text-align: right">
6.67</p>

   </td>
   <td><p style="text-align: right">
6.67</p>

   </td>
   <td><p style="text-align: right">
5.67</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>2.remarketing-remarketing_size:200000-universe_size:1000000
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
1.00</p>

   </td>
   <td><p style="text-align: right">
2.67</p>

   </td>
   <td><p style="text-align: right">
6.17</p>

   </td>
   <td><p style="text-align: right">
6.17</p>

   </td>
   <td><p style="text-align: right">
5.50</p>

   </td>
   <td><p style="text-align: right">
2.83</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>3a.exponential_bow-user_activity_association:independent-universe_size:1000000
   </td>
   <td><p style="text-align: right">
19.67</p>

   </td>
   <td><p style="text-align: right">
1.00</p>

   </td>
   <td><p style="text-align: right">
2.33</p>

   </td>
   <td><p style="text-align: right">
6.00</p>

   </td>
   <td><p style="text-align: right">
6.67</p>

   </td>
   <td><p style="text-align: right">
5.50</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>3b.exponential_bow-user_activity_association:identical-universe_size:1000000
   </td>
   <td><p style="text-align: right">
11.33</p>

   </td>
   <td><p style="text-align: right">
1.00</p>

   </td>
   <td><p style="text-align: right">
2.83</p>

   </td>
   <td><p style="text-align: right">
6.67</p>

   </td>
   <td><p style="text-align: right">
6.50</p>

   </td>
   <td><p style="text-align: right">
6.00</p>

   </td>
   <td><p style="text-align: right">
9.00</p>

   </td>
   <td><p style="text-align: right">
11.50</p>

   </td>
  </tr>
  <tr>
   <td>4a.fully_overlapped-universe_size:1000000-num_sets:20
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
2.00</p>

   </td>
   <td><p style="text-align: right">
4.00</p>

   </td>
   <td><p style="text-align: right">
4.00</p>

   </td>
   <td><p style="text-align: right">
3.50</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>4b.subset-universe_size:1000000-order:random
   </td>
   <td><p style="text-align: right">
8.67</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
1.33</p>

   </td>
   <td><p style="text-align: right">
2.67</p>

   </td>
   <td><p style="text-align: right">
2.67</p>

   </td>
   <td><p style="text-align: right">
2.33</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
8.67</p>

   </td>
  </tr>
  <tr>
   <td>5a.sequentially_correlated-order:random-correlated_sets:one
   </td>
   <td><p style="text-align: right">
13.44</p>

   </td>
   <td><p style="text-align: right">
0.67</p>

   </td>
   <td><p style="text-align: right">
2.89</p>

   </td>
   <td><p style="text-align: right">
6.56</p>

   </td>
   <td><p style="text-align: right">
6.39</p>

   </td>
   <td><p style="text-align: right">
5.81</p>

   </td>
   <td><p style="text-align: right">
0.08</p>

   </td>
   <td><p style="text-align: right">
13.08</p>

   </td>
  </tr>
  <tr>
   <td>5b.sequentially_correlated-order:random-correlated_sets:all
   </td>
   <td><p style="text-align: right">
13.56</p>

   </td>
   <td><p style="text-align: right">
0.69</p>

   </td>
   <td><p style="text-align: right">
3.14</p>

   </td>
   <td><p style="text-align: right">
6.56</p>

   </td>
   <td><p style="text-align: right">
6.67</p>

   </td>
   <td><p style="text-align: right">
6.08</p>

   </td>
   <td><p style="text-align: right">
3.22</p>

   </td>
   <td><p style="text-align: right">
14.92</p>

   </td>
  </tr>
</table>

<br>

It is useful to determine how far epsilon can be reduced as in practice using a lower value for epsilon would conserve the overall privacy budget. For this reason we also ran several simulations with `epsilon = log(3) / 4`. In this setting Bloom filters, regardless of distribution, cannot estimate the cardinality of a single sketch under the 95/10 criteria. On the other hand, Vector of Counts continues to perform reasonably well, and is able to estimate about 10 sets in the best case, however it continues to suffer bias when inputs are correlated.

<br>

<table>
  <tr>
   <td><em>95/10 criteria</em>
   </td>
   <td>Exp BF 2
   </td>
   <td>Geo BF
   </td>
   <td>Cond. Ind.
   </td>
   <td>VoC
   </td>
  </tr>
  <tr>
   <td>1.independent
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
  <tr>
   <td>2.remarketing
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
2.83</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
  <tr>
   <td>3a.exponential_bow-user_activity_association:independent
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
  <tr>
   <td>3b.exponential_bow-user_activity_association:identical
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
9.00</p>

   </td>
   <td><p style="text-align: right">
2.50</p>

   </td>
  </tr>
  <tr>
   <td>4a.fully_overlapped
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
  <tr>
   <td>4b.subset
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
6.67</p>

   </td>
  </tr>
  <tr>
   <td>5a.sequentially_correlated-order:random-correlated_sets:one
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.08</p>

   </td>
   <td><p style="text-align: right">
5.11</p>

   </td>
  </tr>
  <tr>
   <td>5b.sequentially_correlated-order:random-correlated_sets:all
   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
0.00</p>

   </td>
   <td><p style="text-align: right">
3.22</p>

   </td>
   <td><p style="text-align: right">
6.61</p>

   </td>
  </tr>
</table>

<br>

#### Frequency Estimation Under Local Privacy Theme

The following frequency estimators were considered.



*   Stratified sketch of Vector of Counts No Clip
*   Stratified sketch of Vector of Counts With Clip

Due to their extremely poor performance in the no privacy theme, we chose to omit stratified sketches of Bloom filters here. That left us with two variants of Vector of Counts. The Stratified VoC with clipping is far more stable than that without clipping at the tail frequency levels, (see Appendix D), and this led us to generate the full set of results only for the clipped VoC. The following table shows the results.

<br>

<table>
  <tr>
   <td>95/10 criteria
<p>
epsilon = ln(3)
   </td>
   <td>Stratified_VoC-5 Clip
   </td>
   <td>Stratified_VoC-15 Clip
   </td>
  </tr>
  <tr>
   <td>1.homogeneous
   </td>
   <td>2.92
   </td>
   <td>1.58
   </td>
  </tr>
  <tr>
   <td>2.heterogeneous
   </td>
   <td>2.58
   </td>
   <td>1.50
   </td>
  </tr>
  <tr>
   <td>3.publisher_constant
   </td>
   <td>6.75
   </td>
   <td>2.25
   </td>
  </tr>
</table>

<br>

From the table, it is easy to see that there is a trade-off between the maximum frequency to estimate and the number of publishers that can be estimated under a given criterion. The stratified VoC can estimate around 3 publishers if the maximum frequency level is 5, yet it goes down to about 2 publishers if we want to estimate frequency up to 15.

Across both reach and frequency estimation, one overall finding in the local theme is that the accuracy of all of the estimators is impacted by the scenario being considered. That is, the estimators are sensitive to the input data distribution. This presents a serious issue for using any of these methods in practice.

<br>

### Global privacy theme

A secure multi party computation (MPC) protocol is required for all candidates in this theme, and unfortunately such a protocol only exists for Bloom filters. Therefore, because we do not have an MPC protocol for combining VoC, it is omitted from this theme.

#### Reach Estimation Under Global Privacy Theme

For reach we considered the following Bloom filters:

*   Exponential Bloom filter (aka LiquidLegions)
*   Geometric Bloom filter
*   Log Bloom filter

The following table summarizes the performance for the 95/5 performance criteria. Note that 20 is the maximum number of sets the simulation attempted to union, and so this result shows that these methods can union at least 20 sets. However, if we consider the theory, there should be no issue scaling this to any number of sets. This is because regardless of the number of sets being unioned, the noise being added is the same. This effect can be seen in the box plot immediately below the table. Notice that the error remains essentially constant as the number of sets increases.

<br>

<table>
  <tr>
   <td><em>AVERAGE of num_estimable_sets</em>
<p>
<em>(95/5 accuracy)</em>
   </td>
   <td colspan="3" ><em>sketch_estimator_formatted</em>
   </td>
  </tr>
  <tr>
   <td><em>scenario_formatted</em>
   </td>
   <td>Exponential Bloom filter
   </td>
   <td>Geometric Bloom filter
   </td>
   <td>Log Bloom filter
   </td>
  </tr>
  <tr>
   <td>1.independent-universe_size:1000000-small_set:10000
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>2.remarketing-remarketing_size:200000-universe_size:1000000
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>3a.exponential_bow-user_activity_association:independent-universe_size:1000000
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>3b.exponential_bow-user_activity_association:identical-universe_size:1000000
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>4a.fully_overlapped-universe_size:1000000-num_sets:20
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>4b.subset-universe_size:1000000-order:random
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>5a.sequentially_correlated-order:random-correlated_sets:one
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
  <tr>
   <td>5b.sequentially_correlated-order:random-correlated_sets:all
   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
   <td><p style="text-align: right">
20.00</p>

   </td>
  </tr>
</table>

<br>

<p align="center">
<img src="image2.png?raw=true">
</p>

<br>


The more interesting question in this theme is how far epsilon can be reduced while maintaining a particular quality standard. The following plot, which considers the exponential Bloom filter, shows three curves, each of which represents a particular performance criterion, 95/5, 90/10 and 80/20. On the x-axis is the audience size that can be estimated for a given epsilon value, which is shown on the y-axis. All conditions that are above and to the right of the curves meet the performance criteria shown by that curve. Observe that as audience size increases the minimum epsilon stabilizes.

<br>

<p align="center">
<img src="image3.png?raw=true">
</p>
<p>
    <em>Fig: Plot of epsilon vs. the minimum audience size required to achieve three different performance criteria. The audience size is log10 scaled and ranges from 1000 to 10 million. All conditions above and to the right of a curve would meet the quality criterion defined by that curve.</em>
</p>


#### Frequency Estimation Under Global Privacy Theme

For frequency we only considered the LiquidLegions sketch, which as mentioned above, is an exponentially distributed Bloom filter with some extra features for enabling frequency estimation.

The reason for not considering other distributions is that the geometric and exponential distributions are essentially equivalent and we have found them to be superior to the log distribution in the local theme. We have also investigated the properties of exponential Bloom filters more deeply and have found that under proper parameterizations they allow for the construction of an efficient MPC protocol capable of estimating reach and frequency according to the performance criteria discussed here. An upcoming publication will discuss these properties and the MPC protocol in detail.

<br>

<table>
  <tr>
   <td>95/5
<p>
epsilon = ln(3)
   </td>
   <td>LiquidLegions - 15
   </td>
  </tr>
  <tr>
   <td>1.homogeneous-universe_size:200000-num_sets:10
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
  <tr>
   <td>2.heterogeneous-universe_size:200000-num_sets:10
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
  <tr>
   <td>3.publisher_constant_frequency-universe_size:200000-num_sets:10
   </td>
   <td><p style="text-align: right">
10.00</p>

   </td>
  </tr>
</table>

<br>

Overall the results for the global privacy theme are outstanding. Liquid Legions estimates frequencies of up to 15 for at least 10 publishers. Here again, the theory predicts this and we do not see any limit with respect to the number of publishers that can be accomodated.

The only missing piece of data for this theme is the cost of running the MPC framework, which will definitely be more expensive than a system that uses techniques from the local privacy theme. Nonetheless, early tests and back of the envelope estimates give cause for hope, and we look forward to following up in detail as more reliable data becomes available.

<br>

## Conclusions and Next Steps

Pending the results of the MPC performance test we provisionally recommend to use the Liquid Legions sketch along with a MPC protocol for estimating cross media reach and frequency.

There are several reasons to support our recommendation. The first is that the accuracy of MPC-based Liquid Legions sketch for reach measurement is invariant to the number of sets being unioned and the distribution of its inputs. Similarly its accuracy for frequency measurement is invariant to the maximum frequency being measured and the number of sets to be unioned. Moreover, it is the only method that can measure frequency across multiple publishers with an acceptable max frequency. In short, it is highly accurate under all reach and frequency scenarios. The superior scalability of LiquidLegions along with a MPC protocol is critical for building a practical cross-media media measurement solution.

Unfortunately, as compared to the locally private methods, this accuracy comes with substantially higher computational costs, a full understanding of which will require an end-to-end synthetic data test at scale, however experience with similar systems and early performance testing done in private give reason to believe that the operational costs will be reasonable. Moreover, an open source reference implementation of the LiquidLegions MPC protocol will be available soon, at which time these initial test results will also be made available and replicated as necessary. Once this is done we recommend proceeding with an end-to-end synthetic data test of the LiquidLegions sketch and its MPC protocol at scale. We also recommend conducting a WFA sponsored peer review of these results as soon as possible.

----
## References

[1] - [Cardinality and Frequency Estimation Evaluation Framework](https://github.com/world-federation-of-advertisers/cardinality_estimation_evaluation_framework/blob/master/doc/cardinality_and_frequency_estimation_evaluation_framework.md)

[2] - [Privacy-Preserving Secure Cardinality Estimation](https://research.google/pubs/pub49177/)

<br>

## End Notes

<sup>1</sup> We chose not to include uniform counting Bloom filters because of their high memory usage, which makes them impractical for any real system. However, some previous research results indicate that they make good frequency estimators under the no privacy and global themes, while under the local theme they suffer from considerable error.

<sup>2</sup> See [[1]](#references) for more detail on the average relative error and average shuffle distance.

<sup>3</sup> This is a primary reason for why counting Bloom filters are not viable under this theme.

<sup>4</sup> The Geometric Bloom filter and the Exponential Bloom filter have equivalent parameterizations.

<br>

## Appendix

<br>

### Appendix A: Detailed Parameterization of the Sketch and Estimator Candidates


<table>
  <tr>
   <td><strong>Sketch</strong>
   </td>
   <td><strong>Abbreviation</strong>
   </td>
   <td><strong>Estimator</strong>
   </td>
   <td><strong>Configuration</strong>
   </td>
   <td><strong>Supports Frequency</strong>
   </td>
  </tr>
  <tr>
   <td>Hyper Log Log ++
   </td>
   <td>HLL++
   </td>
   <td>
   </td>
   <td>length=16384
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Reach
   </td>
   <td>Cond. Ind.
   </td>
   <td>Conditional independence model
   </td>
   <td>universe size = 1M
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Geometric Bloom Filters[^4]
   </td>
   <td>Geo BF
   </td>
   <td>First Moment Estimator
   </td>
   <td>length = 250K
<p>
probability = 0.000008
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Exponential Bloom Filters
   </td>
   <td>Exp BF 2
   </td>
   <td>First Moment Estimator
   </td>
   <td>length = 250K
<p>
decay_rate = 2
<p>
(equivalent to Geo BF config)
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Exponential Bloom Filter
   </td>
   <td>Exp BF 10
   </td>
   <td>First Moment Estimator
   </td>
   <td>length = 250K
<p>
decay_rate = 10
<p>
(decay rate > 10 known to be better for larger audiences)
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Logarithmic Bloom Filter
   </td>
   <td>Log BF
   </td>
   <td>First Moment Estimator
   </td>
   <td>length = 250K
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Vector-of-Counts
   </td>
   <td>VoC
   </td>
   <td>Sequential estimator
   </td>
   <td>length = 4096
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Meta-VoC Uniform Bloom filter
   </td>
   <td>Meta VoC Uni
   </td>
   <td>Sequential estimator
   </td>
   <td>VoC length = 4096
<p>
Bloom filter length = 5,000,000
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Meta-VoC Exponential Bloom filter
   </td>
   <td>Meta VoC Exp
   </td>
   <td>Sequential estimator
   </td>
   <td>VoC length = 4096
<p>
Bloom filter length = 100,000
<p>
Decay rate = 10
   </td>
   <td>no
   </td>
  </tr>
  <tr>
   <td>Stratified Sketch Exponential Bloom filter
   </td>
   <td>Strat BF
   </td>
   <td>First Moment Estimator
   </td>
   <td>same as Exp BF
   </td>
   <td>yes
   </td>
  </tr>
  <tr>
   <td>Stratified Sketch VoC
   </td>
   <td>Strat VoC
   </td>
   <td>Sequential estimator no-clip
   </td>
   <td>same as VoC
   </td>
   <td>yes
   </td>
  </tr>
  <tr>
   <td>Stratified Sketch VoC
   </td>
   <td>Strat VoC Clip
   </td>
   <td>Sequential estimator with clipping
   </td>
   <td>same as VoC
   </td>
   <td>yes
   </td>
  </tr>
  <tr>
   <td>Liquid Legions
   </td>
   <td>LL
   </td>
   <td>First Moment Estimator
   </td>
   <td>Parameterized as Exp BF, but with counters and same key aggregator
   </td>
   <td>yes
   </td>
  </tr>
</table>

<br>

### Appendix B: Example plots showing VoC has bias when the sets have non-homogeneous correlation:

<br>

<p align="center">
<img src="image4.png?raw=true">
</p>

<br>

<p align="center">
<img src="image5.png?raw=true">
</p>

<br>

### Appendix C: ADBF sketch intersection operation bias

The bias of the stratified ADBF is from the intersection sketch operator, and may very likely come from the difference operator as well. If we use the expectation sketch operator, then the estimated intersection of **two** sketches is biased (under-estimate by ~ 20%):

<br>

<p align="center">
<img src="image6.png?raw=true">
</p>

<br>

If we use the Bayesian sketch operator, then the estimated intersection of **three** sketches is biased (over-estimate by ~ 10%):

<br>

<p align="center">
<img src="image7.png?raw=true">
</p>

<br>

### Appendix D: Stratified VoC with clipping vs without clipping

Without clipping, the variance of the Stratified VoC will blow up when the true cardinality of the given frequency level is low. The following plot shows that the estimated cardinality of the tail frequency levels all deviate away from the truth.

<br>

<p align="center">
<img src="image8.png?raw=true">
</p>

<br>

On the other hand, with clipping, empty sketches are very likely to be clipped, and so the noise associated with them will be less likely to accumulate. Note that clipping may introduce bias. Yet considering the bias and variance trade-off, it is still safe to say that the Stratified VoC with clipping is preferred over the variant that does not use clipping.

<br>

<p align="center">
<img src="image9.png?raw=true">
</p>

<br>

### Appendix E: Boxplots of number of sets _vs_ relative error.

<br>

#### Global DP theme

<br>

Exp-ADBF (decay rate = 2)

<p align="center">
<img src="image10.png?raw=true">
</p>

<br>

Geo-ADBF

<p align="center">
<img src="image11.png?raw=true">
</p>

<br>

Log ADBF

<p align="center">
<img src="image12.png?raw=true">
</p>

<br>

#### Local DP theme

<br>

Exp-ADBF

<p align="center">
<img src="image13.png?raw=true">
</p>

<br>

Geo-ADBF

<p align="center">
<img src="image14.png?raw=true">
</p>

<br>

Log ADBF

<p align="center">
<img src="image15.png?raw=true">
</p>

<br>

VoC

<p align="center">
<img src="image16.png?raw=true">
</p>

<br>

### Appendix F: Bloom filters sensitive to input set size

The Bloom Filters will suffer when the true set size is small, and hence low signal-to-noise ratio it will be. The following plots shows two Exp BF’s performance when all sets are small vs when all sets are large.

<br>

All small

<p align="center">
<img src="image17.png?raw=true">
</p>

<br>

All large

<p align="center">
<img src="image18.png?raw=true">
</p>

<br>

### Appendix G: Some Thoughts on Privacy Budgets

In practice a final consideration that applies across both the local and global theme is that of privacy budget management. This section does not offer any conclusion, only an exploration of some of the issues involved. The first of these is determining the unit of privacy, which starts with deciding whether privacy should be protected for a single impression or a single user. Here, given our previous work, the user is the clear choice.

The next choice is the unit of time over which the privacy budget applies. At one extreme the window could be the lifetime of the user, and at another it could be set to just a few seconds. The former is as inflexible as the latter is a farce. For our purposes here, a single day seems like a reasonable compromise.

Finally it needs to be determined to whom the privacy budget applies. Should each campaign have its own privacy budget, the advertiser (i.e. the entity making queries), or should all advertisers share the same budget? If the budget is per campaign then a single advertiser might be able to determine that the same user had seen many of its campaigns, whereas if the budget is accounted for across advertisers there will simply not be enough budget. The reasonable conclusion is to enforce a budget per advertiser, though admittedly the relationship between advertisers and agencies complicates this, but the basic idea would be to have a budget per entity making queries.

Following this logic, the privacy budget must be split across campaigns for a single advertiser. Note that this consideration makes the local methods completely infeasible and draws some concern to the global methods, which as investigated, can probably only support about 1000 queries per day per advertiser. The good news is that existing research points to a way to allow about 100,000 queries per advertiser per day, which should be sufficient.

Additional research and discussion is required to determine exactly how privacy budgets will be defined and managed.

<br>

### Appendix H: Contributors

#### MRC
* Ron Pinelli Jr

#### Facebook Inc.
* Aaron An
* Miao Yu
* Zeke Huang
* Sanjay Saravanan
* Isabella Ni

#### Google LLC
* Craig Wright
* Xichen Huang
* Sheng Ma
* Jiayu Peng
* Matthew Clegg
* Joey Knightbrook
* Ugur Akyol
* Evgeny Skvortsov
* Raimundo Mirisola
* Preston Lee
