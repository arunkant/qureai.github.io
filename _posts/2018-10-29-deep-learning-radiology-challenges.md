---
layout: post
title: Challenges of Development & Validation of Deep Learning for Radiology
author: Sasank Chilamkurthy
updated: 2018-10-29 12:00:00 +0530
categories:
twitter_image: "https://blog.qure.ai/assets/images/head_ct_study/windows.png"
---

We have recently published an <a href="https://www.thelancet.com/journals/lancet/article/PIIS0140-6736(18)31645-3/fulltext">article</a> on our deep learning algorithms for Head CT in *The Lancet*. This article is the first ever AI in medical imaging paper to be published in this journal.
We described development and validation of these algorithms in the article.
In this blog, I explain some of the challenges we faced in this process and how we solved them. The challenges I describe are fairly general and should be applicable to any research involving AI and radiology images.


### Development

#### 3D Images

First challenge we faced in the development process is that CT scans are three dimensional (3D). There is plethora of research for two dimensional (2D) images, but far less for 3D images. You might ask, why not simply use 3D convolutional neural networks (CNNs) in place of 2D CNNs? Notwithstanding the computational and memory requirements of 3D CNNs, they have been <a href="https://arxiv.org/abs/1705.07750">shown</a> to be inferior to 2D CNN based approaches on a similar problem (action recognition).

So how do we solve it? We need not invent the wheel from scratch when there is a [lot of literature on a similar problem, *action recognition*](https://blog.qure.ai/notes/deep-learning-for-videos-action-recognition-review). Action recognition is classification of action that is present in a given video.
Why is action recognition similar to 3D volume classification? Well, temporal dimension in videos is analogous to the Z dimension in the CT.


<div style="overflow: auto">
    <div style="float: left" id='volume'>
    </div>
    <div id='video' style="float: right">
    </div>

</div>
<p align="center" class="caption">Left: Example Head CT scan. Right: Example video from a <a href="http://www.wisdom.weizmann.ac.il/%7Evision/SpaceTimeActions.html">action recognition dataset</a>. Z dimension in the CT volume is analogous to time dimension in the video.</p>

We have taken a <a href="https://arxiv.org/abs/1406.2199">foundational work</a> from action recognition literature and modified it to our purposes. Our modification was that we have incorporated slice (or frame in videos) level labels in to the network. This is because action recognition literature had a comfort of using pretrained 2D CNNs which we do not share.

#### High Resolution

Second challenge was that CT is of high resolution both spatially and in bit depth. We just downsample the CT to a standard pixel spacing. How about bit depth? Deep learning doesn't work great with the data which is not normalized to [-1, 1] or [0, 1]. We solved this with what a radiologist would use - *windowing*. Windowing is restriction of dynamic range to a certain interval (eg. [0, 80]) and then normalizing it. We applied three windows and passed them as channels to the CNNs.
<p align="center">
    <img src="/assets/images/head_ct_study/windows.png" alt="Windows: brain, blood/subdural and bone">
    <br>
    <small class="caption">Windows: brain, blood/subdural and bone</small>
</p>

This approach allows for multi-class effects to be accounted by the model. For example, a large scalp hemotoma visible in brain window might indicate a fracture underneath it. Conversely, a fracture visible in the bone window is usually correlated with an extra-axial bleed.


#### Other Challenges

There are few other challenges that deserve mention as well:

1. Class Imbalance: We solved the class imbalance issue by weighted sampling and loss weighting.
2. Lack of pretraining: There's no pretrained model like imagenet available for medical images. We found that using imagenet weights actually hurts the performance.


### Validation

Once the algorithms were developed, validation was not without its challenges as well. 
Here are the key questions we started with: does our algorithms generalize well to CT scans not in the development dataset?
Does the algorithm also generalize to CT scans from a different source altogether? How does it compare to radiologists without access to clinical history?

#### Low prevalences and statistical confidence

The validation looks simple enough: just acquire scans (from a different source), get it read by radiologists and compare their reads with the algorithms'.
But statistical design is a challenge! This is because prevalence of abnormalities tend to be low; it can be as low as 1% for some abnormalities. Our key metrics for evaluating the algorithms are sensitivity & specificity and AUC depending on the both. Sensitivity is the trouble maker: we have to ensure there are enough positives in the dataset to ensure narrow enough 95% confidence intervals (CI). Required number of positive scans turns out to be ~80 for a CI of +/- 10% at an expected sensitivity of 0.7.

If we were to chose a randomly sampled dataset, number of scans to be read is ~ 80/prevalence rate = 8000. Suppose there are three readers per scan, number of total reads are 8k * 3 =  24k. So, this is a prohibitively large dataset to get read by radiologists. We cannot therefore have a randomly sampled dataset; we have to somehow enrich the number of positives in the dataset.

#### Enrichment

To enrich a dataset with positives, we have to find the positives from all the scans available. It's like searching for a needle in a haystack. Fortunately, all the scans usually have a clinical report associated with them. So we just have to read the reports and choose the positive reports. Even better, have [an NLP algorithm parse the reports](https://blog.qure.ai/notes/teaching-machines-read-radiology-reports) and randomly sample the required number of positives. We chose this path.

We collected the dataset in two batches, B1 & B2. B1 was all the head CT scans acquired in a month and B2 was the algorithmically selected dataset. So, B1 mostly contained negatives while B2 contained lot of positives. This approach removed any selection bias that might have been present if the scans were manually picked. For example, if positive scans were to be picked by manual & cursory glances at the scans themselves, subtle positive findings would have been missing from the dataset.

<div id="container">
    <canvas id="canvas"></canvas>
</div>
<p align="center" class="caption">Prevalences of the findings in batches B1 and B2. Observe the low prevalences of findings in uniformly sampled batch B1.</p>

#### Reading

We called this enriched dataset, **CQ500 dataset** (C for <a href="http://caring-mi.com">CARING</a> and Q for <a href="http://qure.ai">Qure.ai</a>). The dataset contained 491 scans after the exclusions. Three radiologists independently read the scans in the dataset and the majority vote is considered the gold standard. We randomized the order of the reads to minimize the recall of follow up scans and to blind the readers to the batches of the dataset.  

We make this dataset and the radiologists' reads <a href="http://headctstudy.qure.ai/dataset">public</a> under CC-BY-NC-SA license. Other researchers can use this dataset to benchmark their algorithms. I think it can also be used for some clinical research like measuring concordance of radiologists on various tasks etc.

In addition to the CQ500 dataset, we validated the algorithms on a much larger randomly sampled dataset, **Qure25k dataset**. Number of scans in this dataset was 21095. Ground truths were clinical radiology reports. We used the [NLP algorithm](https://blog.qure.ai/notes/teaching-machines-read-radiology-reports) to get structured data from the reports. This dataset satisfies the statistical requirements, but each scan is read only by a single radiologist who had access to clinical history.

### Results

<table class="table">
    <thead class="font-weight-bold">
      <tr>
        <th>Finding </th>
        <th>CQ500 <small>(95% CI)</small></th>
        <th>Qure25k <small>(95% CI)</small></th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Intracranial hemorrhage</td>
        <td>0.9419 <small>(0.9187-0.9651)</small></td>
        <td>0.9194 <small>(0.9119-0.9269)</small></td>
      </tr>
      <tr>
        <td><span style="margin-left: 1rem">Intraparenchymal</span></td>
        <td>0.9544 <small>(0.9293-0.9795)</small></td>
        <td>0.8977 <small>(0.8884-0.9069)</small></td>
      </tr>
      <tr>
        <td><span style="margin-left: 1rem">Intraventricular</span></td>
        <td>0.9310 <small>(0.8654-0.9965)</small></td>
        <td>0.9559 <small>(0.9424-0.9694)</small></td>
      </tr>
      <tr>
        <td><span style="margin-left: 1rem">Subdural</span></td>
        <td>0.9521 <small>(0.9117-0.9925)</small></td>
        <td>0.9161 <small>(0.9001-0.9321)</small></td>
      </tr>
      <tr>
        <td><span style="margin-left: 1rem">Extradural</span></td>
        <td>0.9731 <small>(0.9113-1.0000)</small></td>
        <td>0.9288 <small>(0.9083-0.9494)</small></td>
      </tr>
      <tr>
        <td><span style="margin-left: 1rem">Subarachnoid</span></td>
        <td>0.9574 <small>(0.9214-0.9934)</small></td>
        <td>0.9044 <small>(0.8882-0.9205)</small></td>
      </tr>
      <tr>
        <td>Calvarial fracture</td>
        <td>0.9624 <small>(0.9204-1.0000)</small></td>
        <td>0.9244 <small>(0.9130-0.9359)</small></td>
      </tr>
      <tr>
        <td>Midline Shift</td>
        <td>0.9697 <small>(0.9403-0.9991)</small></td>
        <td>0.9276 <small>(0.9139-0.9413)</small></td>
      </tr>
      <tr>
        <td>Mass Effect</td>
        <td>0.9216 <small>(0.8883-0.9548)</small></td>
        <td>0.8583 <small>(0.8462-0.8703)</small></td>
      </tr>
    </tbody>
</table>
<p class="caption">AUCs of the algorithms on the both datasets.</p>

Above table shows AUCs of the algorithms on the two datasets. Note that the AUCs are directly comparable. This is because AUC is prevalence independent. AUCs on CQ500 dataset are generally better than that on the Qure25k dataset. This might be because:

<ol>
<li>Ground truths in the Qure25k dataset incorporated clinical information not available to the algorithms and therefore the algorithms did not perform well.</li>
<li>Majority vote of three reads is a better ground truth than that of a single read.</li>
</ol>

<p align="center">
    <img src="/assets/images/head_ct_study/ROCs.png" alt="ROC curves">
    <br>
    <p class="caption">ROC curves for the algorithms on the Qure25k (blue) and CQ500 (red) datasets. TPR and FPR of radiologists are also plotted.</p>
</p>

Shown above is ROC curves on both the datasets. Readers' TPR and FPR are also plotted. We observe that radiologists are either highly sensitive or specific to a particular finding. The algorithms are still yet to beat radiologists, on this task at least! But these should nonetheless be useful to triage or notify physicians.

<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.7.2/Chart.bundle.min.js"></script>
<script type="text/javascript" src="/assets/js/ImageStack.js"></script>
<script type="text/javascript">
    var imageList = getImageList('/assets/images/head_ct_study/stacks/QURE-3', 30);
    var stack = new ImageStack({
    images: imageList,
    height: '15rem',
    width: '15rem'
    });
    $('#volume').append(stack);

    var imageList = getImageList('/assets/images/head_ct_study/stacks/denis_walk_avi', 28);
    var stack = new ImageStack({
    images: imageList,
    height: '15rem',
    width: '20rem'
    });
    $('#video').append(stack);
</script>
<script>
        var barChartData = {
            labels: ['Intracranial hemorrhage', 'Intraparenchymal', 'Intraventricular', 'Subdural', 'Extradural', 'Subarachnoid', 'Fractures', 'Calvarial Fractures', 'Midline Shift', 'Mass effect'],
            datasets: [{
                label: 'Batch B1',
                backgroundColor: 'rgba(255, 99, 132,0.5)',
                borderColor: 'rgb(255, 99, 132)',
                borderWidth: 1,
                data: [
                    16.36,
                    13.55,
                    3.27,
                    4.21,
                    0.93,
                    4.21,
                    3.74,
                    2.80,
                    8.41,
                    13.08,
                ]
            }, {
                label: 'Batch B2',
                backgroundColor: 'rgba(54, 162, 235, 0.5)',
                borderColor: 'rgb(54, 162, 235)',
                borderWidth: 1,
                data: [
                    61.37,
                    37.91,
                    7.58,
                    15.88,
                    3.97,
                    18.41,
                    11.19,
                    10.11,
                    16.97,
                    35.74,
                ]
            }]
        };

        window.onload = function() {
            var ctx = document.getElementById('canvas').getContext('2d');
            window.myHorizontalBar = new Chart(ctx, {
                type: 'horizontalBar',
                data: barChartData,
                options: {
                    elements: {
                        rectangle: {
                            borderWidth: 2,
                        }
                    },
                    responsive: true,
                    legend: {
                        position: 'right',
                    },
                    title: {
                        display: true,
                        text: 'CQ500 Dataset: Prevalences'
                    },
                    scales: {
                        xAxes: [{
                            scaleLabel: {
                                display: true,
                                labelString: 'Prevalance (%)'
                            },
                            display: true,
                            ticks: {
                                steps: 10,
                                stepValue: 10,
                                max: 100
                            }
                        }]
                    },
                    animation:{
                        duration: 0
                    }
                }
            });
        };

</script>

<style type="text/css">
    /*Scroll Stuff*/
    .custom-scroll{
      float: none;
      margin: 0 auto;
    }

    .custom-scroll::-webkit-scrollbar-track
    {
      -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,0.3);
      border-radius: 5px;
      background-color: #F5F5F5;
    }

    .custom-scroll::-webkit-scrollbar
    {
      width: 12px;
      background-color: #F5F5F5;
    }

    .custom-scroll::-webkit-scrollbar-thumb
    {
      border-radius: 5px;
      -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,.3);
      background-color: #464646;
    }

    td{
        word-wrap: break-word;
        hyphens: auto;
    }
</style>