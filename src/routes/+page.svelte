<script>
    import { onMount } from "svelte";
    import { WORKOUTS, SCHEMES, EXERCISES, TEMPOS } from "../utils/swoldier";
    import { clickOutside } from "../utils/clickOutside";

    let workout;
    let muscles = [];
    let goal = 50;
    let showSelectMuscles = false;
    let formulating = false;
    let formultion = null;
    let showExerciseDescription = [];
    let setsCompleted = {};
    let error = false;
    let email = "";
    let subscribed = false;
    let subscribing = false;

    function handleOnChange(event) {
        console.log(event.target.value);
        //save value to firestore
    }

    function handleClickOutside(event) {
        if (!showSelectMuscles) {
            return;
        }
        showSelectMuscles = false;
    }

    async function subscribe() {
        if (subscribed || subscribing || !email) {
            return;
        }
        subscribing = true;
        try {
            const res = await fetch(
                "https://mailing-list-3hzb.onrender.com/api/subscribe",
                {
                    method: "POST",
                    headers: {
                        "Content-type": "application/json",
                    },
                    body: JSON.stringify({
                        platform: "swolenormous",
                        email,
                        password: "80085",
                    }),
                }
            );
            email = "";
            subscribed = true;
        } catch (err) {
            console.log("Failed to subscribe", err);
        } finally {
            subscribing = false;
        }
    }

    onMount(() => {
        if (localStorage.getItem("swolenormous")) {
            formultion = JSON.parse(
                localStorage.getItem("swolenormous")
            ).formultion;
        }
    });

    $: {
        if (workout) {
            muscles = [];
            showSelectMuscles = false;
        }
    }

    //functions
    function shuffleArray(array) {
        for (let i = array.length - 1; i > 0; i--) {
            let j = Math.floor(Math.random() * (i + 1));
            let temp = array[i];
            array[i] = array[j];
            array[j] = temp;
        }
        return array;
    }

    function exercisesFlattener(exercisesObj) {
        //flattens all the variations, creates new combined descriptions (normal first, variant second), and then adds each of the variations to the substitutions array
        const flattenedObj = {};
        for (const [key, val] of Object.entries(exercisesObj)) {
            if (!("variants" in val)) {
                flattenedObj[key] = val;
            } else {
                for (const variant in val.variants) {
                    let variantName = variant + "_" + key;
                    let variantSubstitutes = Object.keys(val.variants)
                        .map((element) => element + " " + key)
                        .filter(
                            (element) =>
                                element.replaceAll(" ", "_") !== variantName
                        );
                    flattenedObj[variantName] = {
                        ...val,
                        description:
                            val.description + "___" + val.variants[variant],
                        substitutes: [
                            ...val.substitutes,
                            ...variantSubstitutes,
                        ].slice(0, 5),
                    };
                }
            }
        }
        return flattenedObj;
    }

    const exercises = exercisesFlattener(EXERCISES);

    async function formulateWorkout() {
        error = false;
        if (formulating || !workout || muscles.length == 0) {
            error = true;
            return;
        }
        formulating = true;
        let exer = Object.keys(exercises);
        exer = exer.filter((key) => exercises[key].meta.environment !== "home");
        let includedTracker = [];
        let numSets = 5;
        let listOfMuscles;

        if (workout === "individual") {
            listOfMuscles = muscles;
        } else {
            listOfMuscles = WORKOUTS[workout][muscles[0]];
        }

        listOfMuscles = new Set(shuffleArray(listOfMuscles));
        let arrOfMuscles = Array.from(listOfMuscles);
        let scheme =
            goal < 33
                ? "strengthPower"
                : goal > 66
                ? "cardiovascularEndurance"
                : "growthHypertrophy";
        let sets = SCHEMES[scheme].ratio
            .reduce((acc, curr, index) => {
                //make this compound and exercise muscle -> array of objects and destructure in loop
                return [
                    ...acc,
                    ...[...Array(parseInt(curr)).keys()].map((val) =>
                        index === 0 ? "compound" : "accessory"
                    ),
                ];
            }, [])
            .reduce((acc, curr, index) => {
                const muscleGroupToUse =
                    index < arrOfMuscles.length
                        ? arrOfMuscles[index]
                        : arrOfMuscles[index % arrOfMuscles.length];
                return [
                    ...acc,
                    {
                        setType: curr,
                        muscleGroup: muscleGroupToUse,
                    },
                ];
            }, []);

        const { compound: compoundExercises, accessory: accessoryExercises } =
            exer.reduce(
                (acc, curr) => {
                    let exerciseHasRequiredMuscle = false;
                    for (const musc of exercises[curr].muscles) {
                        if (listOfMuscles.has(musc)) {
                            exerciseHasRequiredMuscle = true;
                        }
                    }
                    return exerciseHasRequiredMuscle
                        ? {
                              ...acc,
                              [exercises[curr].type]: {
                                  ...acc[exercises[curr].type],
                                  [curr]: exercises[curr],
                              },
                          }
                        : acc;
                },
                { compound: {}, accessory: {} }
            );

        const genWOD = sets.map(({ setType, muscleGroup }) => {
            const data =
                setType === "compound" ? compoundExercises : accessoryExercises;
            const filteredObj = Object.keys(data).reduce((acc, curr) => {
                if (
                    includedTracker.includes(curr) ||
                    !data[curr].muscles.includes(muscleGroup)
                ) {
                    // if (includedTracker.includes(curr)) { console.log('banana', curr) }
                    return acc;
                }
                return { ...acc, [curr]: exercises[curr] };
            }, {});
            const filteredDataList = Object.keys(filteredObj);
            const filteredOppList = Object.keys(
                setType === "compound" ? accessoryExercises : compoundExercises
            ).filter((val) => !includedTracker.includes(val));

            let randomExercise =
                filteredDataList[
                    Math.floor(Math.random() * filteredDataList.length)
                ] ||
                filteredOppList[
                    Math.floor(Math.random() * filteredOppList.length)
                ];

            // console.log(randomExercise)

            if (!randomExercise) {
                return {};
            }

            let repsOrDuraction =
                exercises[randomExercise].unit === "reps"
                    ? Math.min(...SCHEMES[scheme].repRanges) +
                      Math.floor(
                          Math.random() *
                              (Math.max(...SCHEMES[scheme].repRanges) -
                                  Math.min(...SCHEMES[scheme].repRanges))
                      ) +
                      (setType === "accessory" ? 4 : 0)
                    : Math.floor(Math.random() * 40) + 20;
            const tempo = TEMPOS[Math.floor(Math.random() * TEMPOS.length)];

            if (exercises[randomExercise].unit === "reps") {
                const tempoSum = tempo
                    .split(" ")
                    .reduce((acc, curr) => acc + parseInt(curr), 0);
                if (tempoSum * parseInt(repsOrDuraction) > 85) {
                    repsOrDuraction = Math.floor(85 / tempoSum);
                }
            } else {
                //set to nearest 5 seconds
                repsOrDuraction = Math.ceil(parseInt(repsOrDuraction) / 5) * 5;
            }
            includedTracker.push(randomExercise);

            return {
                name: randomExercise,
                tempo,
                rest: SCHEMES[scheme]["rest"][setType === "compound" ? 0 : 1],
                reps: repsOrDuraction,
                ...exercises[randomExercise],
            };
        });

        formultion = genWOD.filter(
            (element) => Object.keys(element).length > 0
        );
        localStorage.setItem("swolenormous", JSON.stringify({ formultion }));
        await new Promise((r) => setTimeout(r, 1000));

        formulating = false;
        console.log(formultion);

        document.getElementById("workout").scrollIntoView({
            behavior: "smooth",
        });
    }
</script>

<main
    class="min-h-screen relative bg-gradient-to-r from-slate-800 to-slate-950 text-white flex flex-col"
>
    <a
        href="#merch"
        class="absolute top-0 right-0 p-2 text-xs sm:text-sm text-slate-700 duration-200 cursor-pointer hover:text-blue-500 uppercase"
    >
        <p>Merch</p>
    </a>
    <!-- <header
        class="py-4 sm:py-6 flex items-center justify-between max-w-[1200px] mx-auto w-full border-b border-solid border-slate-600"
    >
        <h1 class="font-medium">Swole<b class="font-medium text-blue-500">normous</b></h1>
        <button
            class="px-4 py-2 rounded-md bg-slate-950 border-[1.5px] border-blue-600 duration-200 hover:border-blue-400 border-solid flex items-center gap-4"
        >
            <p>Train</p>
            <i class="fa-solid fa-dumbbell" />
        </button>
    </header> -->
    <section
        class="flex flex-col max-w-[1200px] w-full mx-auto p-6 min-h-screen justify-center items-center gap-10 sm:gap-12 md:gap-14"
    >
        <div class="flex flex-col text-center sm:gap-1 md:gap-2">
            <p class="uppercase font-medium text-lg">It's time to get</p>
            <h1 class="text-4xl sm:text-5xl md:text-6xl font-semibold">
                SWOLE<b class="font-semibold text-blue-400">NORMOUS</b>
            </h1>
        </div>
        <p
            class="text-slate-200 text-sm sm:text-base mx-auto max-w-[700px] w-full text-center"
        >
            I hereby acknowledgement that I may become <b
                class="text-blue-400 font-medium">unbelievably swolenormous</b
            >
            and accept all risks of becoming the local
            <b class="text-blue-400 font-medium">mass montrosity</b>, afflicted
            with severe body dismorphia, unable to fit through doors.
        </p>
        <a
            href="#generate"
            class=" relative group duration-200 blueShadow font-medium p-[1.5px] overflow-hidden rounded-md"
        >
            <div
                class="absolute w-[110%] aspect-square top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 z-0"
            >
                <div
                    class="absolute inset-0 bg-gradient-to-r from-blue-400 to-cyan-400 z-0 spinAnimation"
                />
            </div>
            <p
                class="bg-slate-950 duration-200 px-4 sm:px-8 py-2 sm:py-4 relative rounded-md z-10"
            >
                Accept & Begin
            </p>
        </a>
    </section>
    <section
        id="generate"
        class="flex flex-col min-h-screen gap-10 sm:gap-12 md:gap-14 pb-24"
    >
        <div
            class="flex flex-col text-center sm:gap-1 py-10 sm:py-14 md:py-20 md:gap-2 bg-slate-950"
        >
            <p class="uppercase font-medium text-lg">Generate your workout</p>
            <h1 class="text-4xl sm:text-5xl md:text-6xl font-semibold">
                It's <b class="font-semibold text-blue-400">HUGE</b> o'clock
            </h1>
        </div>
        <div
            class="flex flex-col gap-8 sm:gap-10 md:gap-14 px-4 w-full sm:w-fit mx-auto max-w-full"
        >
            <div class="flex flex-col gap-3 sm:gap-4">
                <div class="flex items-end gap-2 mx-auto sm:gap-4">
                    <p
                        class="text-3xl sm:text-4xl md:text-5xl font-semibold text-slate-400"
                    >
                        01
                    </p>
                    <div class="flex items-center gap-2 pb-0.5 sm:pb-1">
                        <p class="text-lg sm:text-2xl md:text-3xl">
                            Pick your poison
                        </p>
                        <!-- <i class="fa-solid fa-skull-crossbones" /> -->
                    </div>
                </div>
                <p class="text-sm sm:text-base mx-auto">
                    Select the workout you wish to endure.
                </p>
                <div
                    class="grid grid-cols-2 sm:grid-cols-4 sm:mx-auto gap-2 sm:gap-4"
                >
                    {#each Object.keys(WORKOUTS) as workoutType, index}
                        <button
                            on:click={() => (workout = workoutType)}
                            class={"capitalize duration-200 p-4 grid bg-slate-950 font-light text-sm sm:text-base rounded-md place-items-center border-solid border-[1.5px] border-solid " +
                                (workout === workoutType
                                    ? "  border-blue-500"
                                    : " border-transparent hover:border-blue-500")}
                        >
                            <p>
                                {workoutType.replace("_", " ")}
                            </p>
                        </button>
                    {/each}
                </div>
            </div>
            <div class="flex flex-col gap-3 sm:gap-4">
                <div class="flex items-end gap-2 mx-auto sm:gap-4">
                    <p
                        class="text-3xl sm:text-4xl md:text-5xl font-semibold text-slate-400"
                    >
                        02
                    </p>
                    <div class="flex items-center gap-2 pb-0.5 sm:pb-1">
                        <p class="text-lg sm:text-2xl md:text-3xl">
                            Lock on targets
                        </p>
                        <!-- <i class="fa-solid fa-skull-crossbones" /> -->
                    </div>
                </div>
                <p class="text-sm sm:text-base mx-auto">
                    Select the muscles judged for annihilation.
                </p>
                <button
                    use:clickOutside
                    on:click_outside={handleClickOutside}
                    on:click={() => {
                        if (!workout) {
                            return;
                        }
                        showSelectMuscles = !showSelectMuscles;
                    }}
                    class={"p-4 bg-slate-950 flex items-center font-light text-sm sm:text-base justify-center gap-2 relative " +
                        (showSelectMuscles
                            ? " rounded-t-md border-b border-solid border-slate-900"
                            : " rounded-md") +
                        (workout ? " " : " cursor-not-allowed")}
                >
                    {#if muscles.length > 0}
                        <p class="capitalize">{muscles.join(" & ")}</p>
                    {:else}
                        <p class="">
                            Select muscle groups {workout === "individual"
                                ? "(up to 3)"
                                : " "}
                        </p>
                    {/if}
                    <div class="absolute top-1/2 right-4 -translate-y-1/2">
                        <i
                            class={"fa-solid fa-caret-down duration-200 " +
                                (showSelectMuscles ? " rotate-180" : " ")}
                        />
                    </div>
                    {#if showSelectMuscles}
                        <div
                            class="absolute top-full left-0 w-full z-20 flex flex-col bg-slate-950 mt-[1px] rounded-b-md overflow-hidden"
                        >
                            {#if workout == "individual"}
                                {#each WORKOUTS[workout] as muscle, muscleIndex}
                                    <button
                                        on:click={(e) => {
                                            e.stopPropagation();
                                            if (muscles.includes(muscle)) {
                                                muscles = muscles.filter(
                                                    (val) => val !== muscle
                                                );
                                                return;
                                            }
                                            if (muscles.length === 3) {
                                                showSelectMuscles = false;
                                                return;
                                            }
                                            muscles = [...muscles, muscle];
                                        }}
                                        class={"capitalize p-2 cursor-pointer duration-200 " +
                                            (muscles.includes(muscle)
                                                ? " text-blue-500"
                                                : " hover:text-blue-500")}
                                    >
                                        <p>
                                            {muscle.replaceAll("_", " ")}
                                        </p>
                                    </button>
                                {/each}
                            {:else}
                                {#each Object.keys(WORKOUTS[workout]) as muscle, muscleIndex}
                                    <button
                                        on:click={(e) => {
                                            e.stopPropagation();
                                            muscles = [muscle];
                                            showSelectMuscles = false;
                                        }}
                                        class={"capitalize p-2 cursor-pointer duration-200 " +
                                            (muscles.includes(muscle)
                                                ? " text-blue-500"
                                                : " hover:text-blue-500")}
                                    >
                                        <p>
                                            {muscle.replaceAll("_", " ")}
                                        </p>
                                    </button>
                                {/each}
                            {/if}
                        </div>
                    {/if}
                </button>
            </div>
            <div class="flex flex-col gap-3 sm:gap-4">
                <div class="flex items-end gap-2 mx-auto sm:gap-4">
                    <p
                        class="text-3xl sm:text-4xl md:text-5xl font-semibold text-slate-400"
                    >
                        03
                    </p>
                    <div class="flex items-center gap-2 pb-0.5 sm:pb-1">
                        <p class="text-lg sm:text-2xl md:text-3xl">
                            Become Juggernaut
                        </p>
                        <!-- <i class="fa-solid fa-skull-crossbones" /> -->
                    </div>
                </div>
                <p class="text-sm sm:text-base mx-auto">
                    Select your ultimate objective.
                </p>
                <div class="grid grid-cols-3 gap-2 sm:gap-3 gap-y-1">
                    {#each Object.keys(SCHEMES) as scheme, index}
                        <p
                            class={"text-xs hidden sm:inline sm:text-sm text-slate-300 " +
                                (index === 1
                                    ? " text-center"
                                    : index === 2
                                    ? " text-right"
                                    : "")}
                        >
                            {scheme === "strengthPower"
                                ? "Strength & Power"
                                : scheme === "growthHypertrophy"
                                ? "Growth & Hypertrophy"
                                : " Cardiovascular & Endurance"}
                        </p>
                        <p
                            class={"text-xs sm:hidden sm:text-sm text-slate-300 " +
                                (index === 1
                                    ? " text-center"
                                    : index === 2
                                    ? " text-right"
                                    : "")}
                        >
                            {scheme === "strengthPower"
                                ? "Strength"
                                : scheme === "growthHypertrophy"
                                ? "Growth "
                                : " Cardio"}
                        </p>
                    {/each}
                    <div
                        class="py-4 bg-slate-950 rounded-md col-span-3 text-slate-950 flex flex-col"
                    >
                        <input
                            bind:value={goal}
                            on:change={handleOnChange}
                            max={100}
                            min={0}
                            type="range"
                            class="w-full"
                        />
                    </div>
                </div>
            </div>
            <!-- <button
                class="py-4 px-8 sm:px-10  rounded-lg border sm:mx-auto border-blue-400 border bg-black blueShadow hover:border-blue-400 duration-200 font-medium"
            >
                Formulate
            </button> -->
            {#if error}
                <p class="mx-auto text-rose-400 text-sm">
                    Please ensure your selections are complete.
                </p>
            {/if}
            <button
                on:click={formulateWorkout}
                class=" relative mx-auto group duration-200 blueShadow font-medium p-[1.5px] overflow-hidden rounded-md"
            >
                <div
                    class="absolute w-[110%] aspect-square top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 z-0"
                >
                    <div
                        class="absolute inset-0 bg-gradient-to-r from-blue-400 to-cyan-400 z-0 spinAnimation"
                    />
                </div>
                <p
                    class="bg-slate-950 duration-200 px-4 sm:px-8 py-2 sm:py-4 relative rounded-md z-10"
                >
                    Formulate
                </p>
                {#if formulating}
                    <div
                        class="absolute z-[15] inset-[1.5px] rounded-md bg-slate-950 grid place-items-center"
                    >
                        <i class="fa-solid fa-spinner animate-spin" />
                    </div>
                {/if}
            </button>
        </div>
    </section>
    {#if formultion}
        <section
            id="workout"
            class="flex flex-col min-h-screen gap-10 sm:gap-12 md:gap-14 pb-24"
        >
            <div
                class="flex flex-col text-center px-4 sm:gap-1 py-10 sm:py-14 md:py-20 md:gap-2 bg-slate-950"
            >
                <p class="uppercase font-medium text-lg">welcome to</p>
                <h1 class="text-4xl sm:text-5xl md:text-6xl font-semibold">
                    The <b class="font-semibold text-blue-400">DANGER</b> zone
                </h1>
            </div>
            <div
                class="mx-auto max-w-[700px] w-fit text-slate-400 text-xs sm:text-sm text-justify px-4"
            >
                *<b class="font-semibold">Note</b> -
                <span class="text-blue-400">reps</span>
                is the number of repetitions,
                <span class="text-blue-400">rest</span>
                is specified in seconds, and
                <span class="text-blue-400">tempo</span>
                is the number of seconds for each movement phase in the order of
                eccentric - isometric - concentric (or down - pause - up).
                <br /><br />For
                <span class="text-blue-400">weight selection</span>, choose a
                weight that allows you to complete the repetitions with minimal
                sacrifice to form.
                <br /><br />Happy lifting!
            </div>
            <div
                class="flex flex-col gap-4 sm:gap-6 md:gap-8 max-w-[900px] w-full mx-auto px-4"
            >
                {#each formultion as exercise, i}
                    <div
                        class="p-4 rounded-md flex flex-col gap-4 bg-slate-950"
                    >
                        <div
                            class="flex flex-col sm:flex-row sm:items-center sm:flex-wrap gap-x-4"
                        >
                            <h4
                                class="text-3xl hidden sm:inline sm:text-4xl md:text-5xl font-semibold text-slate-400"
                            >
                                0{i + 1}
                            </h4>
                            <h2
                                class="capitalize whitespace-nowrap truncate max-w-full text-lg sm:text-xl md:text-2xl flex-1 md:text-center"
                            >
                                {exercise.name.replaceAll("_", " ")}
                            </h2>
                            <p class="text-sm text-slate-400 capitalize">
                                {exercise.type}
                            </p>
                        </div>

                        <div class="flex flex-col">
                            <h3 class="text-slate-400 text-sm">
                                Muscle groups
                            </h3>
                            <p class="capitalize">
                                {exercise.muscles.join(" & ")}
                            </p>
                        </div>
                        <div
                            class="grid grid-cols-2 sm:grid-cols-4 sm:place-items-center gap-2"
                        >
                            {#each ["reps", "rest", "tempo"] as info}
                                <div
                                    class="flex flex-col p-2 rounded border-[1.5px] border-solid border-slate-900 w-full"
                                >
                                    <h3
                                        class="text-slate-400 text-sm capitalize"
                                    >
                                        {info === "reps"
                                            ? `${exercise.unit}`
                                            : info}
                                    </h3>

                                    <p class="font-medium">{exercise[info]}</p>
                                </div>
                            {/each}
                            <button
                                class="flex flex-col p-2 rounded border-[1.5px] duration-200 border-solid border-blue-900 hover:border-blue-600 w-full duration-200"
                                on:click={() => {
                                    if (!(i in setsCompleted)) {
                                        setsCompleted[i] = 1;
                                        return;
                                    }
                                    if (setsCompleted[i] === 5) {
                                        setsCompleted[i] = 0;
                                        return;
                                    }
                                    setsCompleted[i] = setsCompleted[i] + 1;
                                }}
                            >
                                <h3 class="text-slate-400 text-sm capitalize">
                                    Sets completed
                                </h3>
                                <p class="font-medium">
                                    {setsCompleted[i] || 0} / 5
                                </p>
                            </button>
                        </div>

                        <div
                            class="flex flex-col cursor-pointer rounded bg-slate-900"
                        >
                            <button
                                on:click={(e) => {
                                    e.stopPropagation();
                                    if (showExerciseDescription.includes(i)) {
                                        showExerciseDescription =
                                            showExerciseDescription.filter(
                                                (val) => val !== i
                                            );
                                        return;
                                    }
                                    showExerciseDescription = [
                                        ...showExerciseDescription,
                                        i,
                                    ];
                                }}
                                class={"flex items-center justify-between gap-2 p-2"}
                            >
                                <p>Description</p>
                                {#if showExerciseDescription.includes(i)}
                                    <i class="fa-solid fa-minus" />
                                {:else}
                                    <i class="fa-solid fa-plus" />
                                {/if}
                            </button>
                            {#if showExerciseDescription.includes(i)}
                                <div class="h-[1px] bg-slate-950" />
                                <ul class="flex flex-col text-sm p-2 gap-1">
                                    {#each exercise.description.split("___") as description}
                                        <li class="list-disc list-inside">
                                            {description}
                                        </li>
                                    {/each}
                                </ul>
                            {/if}
                        </div>
                    </div>
                {/each}
                <div class="flex items-center gap-4 justify-between mt-8">
                    <!-- <button
                        class="border-[1.5px] border-solid border-blue-600 hover:opacity-60 duration-200 px-8 py-4 rounded-md"
                        >Reset</button
                    > -->
                    <a
                        href="#generate"
                        class=" relative text-center mx-auto group duration-200 blueShadow font-medium p-[1.5px] overflow-hidden rounded-md"
                    >
                        <div
                            class="absolute w-[110%] aspect-square top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 z-0"
                        >
                            <div
                                class="absolute inset-0 bg-gradient-to-r from-blue-400 to-cyan-400 z-0 spinAnimation"
                            />
                        </div>
                        <p
                            class="bg-slate-950 duration-200 px-4 sm:px-8 py-2 sm:py-4 relative rounded-md z-10"
                        >
                            Start Over
                        </p>
                    </a>
                </div>
            </div>
        </section>
    {/if}

    <div
        id="merch"
        class="flex flex-col text-center px-4 gap-6 sm:gap-10 py-10 sm:py-14 md:py-20 bg-slate-950"
    >
        <div
            class="flex flex-col text-center sm:gap-1 py-10  md:gap-2 bg-slate-950"
        >
            <p class="uppercase font-medium text-lg">Look fly while you train</p>
            <h1 class="text-4xl sm:text-5xl md:text-6xl font-semibold">
                Tasteful <b class="font-semibold text-blue-400">GYM</b> tees
            </h1>
        </div>
        <div
            class="flex items-stretch sm:grid sm:grid-cols-2 snap-x snap-mandatory overflow-auto sm:overflow-none gap-3 sm:gap-4 w-full max-w-[600px] mx-auto"
        >
            <a
                href={"https://www.teepublic.com/t-shirt/47312139-its-time-to-get-swolenormous-dark"}
                target="_blank"
                class="cursor-pointer w-full shrink-0 snap-center overflow-hidden"
            >
                <img
                    src="https://i.imgur.com/0zZCaZt.png"
                    alt="dark-tshirt"
                    class="w-full object-cover sm:hover:scale-[1.05]"
                />
            </a>
            <a
                href={"https://www.teepublic.com/t-shirt/47312141-its-time-to-get-swolenormous"}
                target="_blank"
                class="cursor-pointer w-full shrink-0 snap-center overflow-hidden"
            >
                <img
                    src="https://i.imgur.com/zbspSDk.png"
                    alt="dark-tshirt"
                    class="w-full object-cover sm:hover:scale-[1.05]"
                />
            </a>
        </div>
    </div>
    <footer
        class="p-4 sm:p-6 py-16 sm:py-24 md:py-32 flex flex-col justify-center items-center gap-4 border-t border-solid border-blue-950"
    >
        <!-- <div class="" -->
        <div class="flex flex-col gap-10 w-fit max-w-[600px] mx-auto">
            <!-- <p class="rounded text-center">
                Hi! I'm <b class="text-blue-400 font-medium">James</b>. This is
                where I make stuff on the web. Obligatory links:
            </p> -->

            <div class="flex flex-col gap-2 w-full">
                <div class="flex items-center gap-2 mx-auto">
                    <p class="text-center font-light">Get new posts</p>
                    <!-- <i class="fa-regular fa-envelope" /> -->
                </div>
                <div
                    class="flex items-stretch rounded-md border-[1.5px] border-blue-400 focus-within:border-blue-600 duration-200"
                >
                    <input
                        type="text"
                        placeholder="Your email"
                        bind:value={email}
                        class="p-2 flex-1 bg-transparent focus:outline-none outline-none w-full max-w-full"
                    />
                    <button
                        on:click={subscribe}
                        class="grid place-items-center px-2 duration-200 hover:text-blue-400 relative"
                    >
                        {#if subscribing}
                            <div
                                class="absolute inset-0 rounded-md flex items-center justify-end pr-2"
                            >
                                <i class="fa-solid fa-spinner animate-spin" />
                            </div>
                        {/if}
                        {#if subscribed}
                            <div
                                class="absolute inset-0 rounded-md flex items-center justify-end pr-2"
                            >
                                <i class="fa-solid fa-check text-green-400" />
                            </div>
                        {/if}
                        <p
                            class={"duration-200 " +
                                (subscribing || subscribed
                                    ? " opacity-0"
                                    : "opacity-100")}
                        >
                            Subscribe
                        </p>
                    </button>
                </div>
            </div>
            <div
                class="flex flex-col w-fit mx-auto items-center justify-center py-2 gap-2"
            >
                <p>By</p>
                <a
                    href="https://smoljames.com"
                    target="_blank"
                    class=" font-semibold text-5xl sm:hover:scale-110 z-10 w-fit flex items-center justify-center rounded-lg"
                >
                    <p>
                        Smol<b class="font-semibold text-blue-400">james</b>
                    </p>
                </a>
            </div>
        </div>
    </footer>
</main>
