<script lang="ts">
    import {PUBLIC_TEACH_TEST_CYCLES, PUBLIC_END_TEST_CYCLES} from '$env/static/public';
    import {ListBox, ListBoxItem} from '@skeletonlabs/skeleton';
    import Header from './Header.svelte';
    import Datapoint from './Datapoint.svelte';
    import type {TInteractiveOrStatic, TTestOrTeaching} from '$lib/types';
    import {base} from "$app/paths";
    import backend from "$lib/backend";
    import FeedbackWindow from "$lib/components/FeedbackWindow.svelte";
    import {createEventDispatcher} from "svelte";
    import '$lib/../global.css';

    const dispatch = createEventDispatcher();

    export let experimentPhase: TTestOrTeaching;
    export let interactiveOrStatic: TInteractiveOrStatic;
    export let feature_names;
    export let prediction_choices;

    export let instance_prediction: string;

    export let datapoint: { [key: string]: string };
    export let feature_tooltips: { [key: string]: string };
    export let feature_units: { [key: string]: string };
    export let prediction_question =
        'What will the model predict for the current case?';

    export let confidence_level = "-1";
    export let feedback = "";

    export let teaching_intro = "";
    if (interactiveOrStatic === 'static') {
        teaching_intro = "Task: <b>Guess what the model will predict</b>, based on the information below. Afterward, you can read the explanations to <b>understand the model's prediction</b>.";
    } else if (interactiveOrStatic === 'interactive') {
        teaching_intro = "Task: <b>Guess what the model will predict</b>, based on the information below. Afterward, you can ask questions to <b>understand the model's prediction</b>.";
    } else { // chat
        teaching_intro = "Task: <b>Guess what the model will predict</b>, based on the information below. Afterward, you can ask questions to <b>understand the model's prediction</b>.";
    }

    export let test_intro =
        "<b>Test your knowledge</b> about the model. <br> What do you think the model will predict for this case? <br> </b> <b>Note</b>: You will not receive \n" +
        "explanations this time. <br> <b><span style=\"color: purple;\">Purple rows</span></b> indicate changes in the attribute values with the old value (the one in the previous training phase) struck through.";

    export let final_test_intro =
        "Final test of your knowledge about the <b>model's decision process</b>.<br> Based on what you learned before, what do you think the <b>model will predict</b> for this case? <br> <b>Note:</b> You will not receive \n" +
        "explanations this time.";

    export let introduction_test_intro =
        "The table shows a <b>description of a person</b>. <br> " +
        "<b>Task</b>: Guess if the current person is earning <b>more or less than 50k$</b> a year.<br>" +
        "<b>Note</b>: You will not see the model prediction and explanations in the Introduction Phase.";

    export let datapoint_count: number | null = null;

    export let user_id: string | null = null;

    // Extract values from prediction_choices dictionary to list and append "I don't know" option
    export let options = Object.values(prediction_choices).concat(["I don't know"]);

    // get event categories
    export let selected_prediction: string | null = null;

    $: {
        if (confidence_level !== "-1" && experimentPhase === 'intro-test') {
            logPrediction();
            dispatch('next');
        }
    }

    function feedbackClicked(event) {
        feedback = event.detail.feedback;
        // Check if feedback and confidence level are set
        console.log("Checking");
        const checks = [
            {condition: confidence_level === "-1", message: 'Please select your confidence before proceeding.'},
            {condition: feedback === "", message: 'Please describe why you made the decision.'}
        ];
        for (let check of checks) {
            if (check.condition) {
                alert(check.message);
                return;
            }
        }
        logPrediction(event);
    }

    async function logPrediction(event?) {
        const isFinalTest = experimentPhase === 'final-test';
        const isIntroTest = experimentPhase === 'intro-test';

        // If intro and confidence level not set, return
        if (isIntroTest && confidence_level === "-1") {
            return;
        }

        if (isFinalTest && confidence_level === "-1") {
            return;
        }

        if ((isFinalTest) && event) {
            feedback = event?.detail?.feedback || '';
            if (feedback === '') {
                return; // Don't log if feedback is empty in final test
            }
        }

        // Dispatch clicked selection event to show loading spinner if not teaching
        if (experimentPhase !== 'teaching') {
            dispatch('clicked');
            console.log("Dispatching clicked event");
        }

        // Extract feedback and confidence level if applicable
        let confidenceLevelToSend = confidence_level;

        // Prepare common details object with optional properties based on the phase
        const details = {
            datapoint_count: datapoint_count,
            prediction: selected_prediction,
            true_label: instance_prediction,
            ...(isIntroTest || isFinalTest) && {confidence_level: confidenceLevelToSend},
            ...(isFinalTest && {feedback: feedback})
        };

        // Log event if not final test
        if (!isFinalTest) {
            await fetch(`${base}/api/log_event`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    user_id: user_id,
                    event_source: experimentPhase,
                    event_type: 'user_prediction',
                    details: details,
                })
            });
        }

        // Check if user_id and selected_prediction are not null and log
        if (user_id !== null && selected_prediction !== null) {
            await backend.xai(user_id).set_user_prediction(selected_prediction);
        }
        // Reset confidence level and dispatch next event if not in teaching.
        if (experimentPhase !== 'teaching' && experimentPhase !== "final-test") {
            confidence_level = "-1";
        }

        if (experimentPhase === "test") {
            dispatch('next');
        }

        if (isFinalTest && confidence_level !== "-1") {
            // Dispatch next event if confidence level is set
            await fetch(`${base}/api/log_event`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    user_id: user_id,
                    event_source: experimentPhase,
                    event_type: 'user_prediction',
                    details: details,
                })
            });
            confidence_level = "-1";
            console.log("Dispatching next event");
            dispatch('next');
            return;
        }
    }
</script>


<div
        class:teaching-color={experimentPhase === 'teaching'}
        class:testing-color={experimentPhase === 'test'}
        class:final-testing-color={experimentPhase === 'final-test'}
        class:intro-testing-color={experimentPhase === 'intro-test'}
        class="inputarea h-full flex-1 overflow-y-auto shadow-[0_15px_15px_-5px_rgba(0,0,0,0.2)] mx-1.5 my-0"
>
    <Header>Task Description</Header>
    <div class="content-align">
        {#if experimentPhase === 'test'}
            <h2 style="text-align: center; color: purple;">Testing Phase</h2>
            <p class="mb-3 centered-text">{@html test_intro}</p>
        {/if}
        {#if experimentPhase === 'teaching'}
            <h2 style="text-align: center; color: green;">Learning Phase</h2>
            <p class="mb-3 text-xs centered-text">{@html teaching_intro}</p>
        {/if}
        {#if experimentPhase === 'final-test'}
            <h2 style="text-align: center; color: dodgerblue;">Final Testing Phase</h2>
            <p class="mb-3 centered-text">{@html final_test_intro}</p>
        {/if}
        {#if experimentPhase === 'intro-test'}
            <h2 style="text-align: center; color: dodgerblue; margin-top: 15px;">Introduction Phase</h2>
            <p class="mb-3 centered-text">{@html introduction_test_intro}</p>
        {/if}
        {#if datapoint_count !== null}
            <h2 style="text-align: center">
                Current Person ({datapoint_count} of
                {#if experimentPhase === 'test'}
                    {PUBLIC_TEACH_TEST_CYCLES}
                {:else if experimentPhase === 'teaching'}
                    {PUBLIC_TEACH_TEST_CYCLES}
                {:else if experimentPhase === 'final-test'}
                    {PUBLIC_END_TEST_CYCLES}
                {:else if experimentPhase === 'intro-test'}
                    {PUBLIC_TEACH_TEST_CYCLES}
                {/if})
            </h2>
        {/if}
    </div>

    <main>
        <Datapoint
                header={['Attribute', 'Value']}
                body={Object.entries(datapoint).map(([key, value]) => {
                // Check if the value is an object with 'current' and 'old' properties
                if (value && typeof value === 'object' && 'current' in value) {
                    // Format the value to include both current and old information
                    let formattedValue = `Current: ${value.current}`;
                    if ('old' in value) {
                        formattedValue += `, Old: ${value.old}`;
                    }
                    return [key, formattedValue];
                } else {
                    // For simple string values or other types that do not require formatting
                    return [key, typeof value === 'string' ? value : JSON.stringify(value)];
                }
            })}
                feature_tooltips={feature_tooltips}
                feature_units={feature_units}
                feature_names={feature_names}
        />
    </main>
    <div class="content-align">
        {#if (experimentPhase === 'final-test' || experimentPhase === 'intro-test') && selected_prediction != null}
            <div style="display: flex; align-items: center;">
                <h2 style="text-align: center; margin-right: 10px; margin-top: 15px;">How confident are you in your
                    choice?</h2>
                <select id="confidence" name="confidence" bind:value={confidence_level} class="inline-feature-select"
                        style="margin-top: 17px; font-size: 17px; padding: 9px; padding-right: 25px;">
                    <option value="-1">-- Select --</option>
                    <option value="0">Not confident at all</option>
                    <option value="1">Slightly confident</option>
                    <option value="2">Somewhat confident</option>
                    <option value="3">Fairly confident</option>
                    <option value="4">Completely confident</option>
                </select>
            </div>
            {#if (experimentPhase === 'final-test')}
                <FeedbackWindow
                        placeholder="Please describe why you made the decision..."
                        submitLabel="Next"
                        on:feedbackSubmit={feedbackClicked}
                />
            {/if}
        {:else}
            <form>
                <div class="mt-6">
                    <h2 style="text-align: center">Make a prediction</h2>
                    <p class="mb-3 center-text text-xs centered-text">{prediction_question}</p>
                    <div class="variant-ghost-surface w-fit mx-auto">
                        <ListBox
                                active="variant-filled-primary"
                                hover="hover:variant-soft-primary"
                                display="flex-col">
                            {#each options as option, index}
                                <ListBoxItem class="text-sm" bind:group={selected_prediction} name="justify"
                                             value={option}
                                             on:click={() => {selected_prediction = option; logPrediction();}}
                                >{option}</ListBoxItem>
                            {/each}
                        </ListBox>
                    </div>
                </div>
            </form>
        {/if}
    </div>
</div>

<style lang="postcss">
    .inputarea {
        background: var(--questions-bg);
        min-height: 97vh;
        max-height: 97vh;
    }

    @media (max-width: 600px) {
        .inputarea {
            min-height: 90vh;
            max-height: 90vh;
        }
    }

    @media (min-width: 601px) and (max-width: 900px) {
        .inputarea {
            min-height: 95vh;
            max-height: 95vh;
        }
    }

    .content-align {
        width: 90%; /* Same width as table */
        margin: 0 auto; /* To center it */
    }

    .testing-color {
        background-color: lightgray;
    }

    .centered-text {
        text-align: center;
    }

</style>
