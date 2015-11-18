A Xenon stateful service moves from one state to another, often based on client input. The service implementation needs to validate the input, taking the current state into account, and update the state. In that regard, you can think of a stateful service as a FSM (Finite State Machine).

Xenon provides a common abstraction for a FSM to facilitate capturing a service' set of states and transitions among states:
* FSM - a interface for a finite state machine. State type and transition type are interface parameters.
* FSMTracker - a class that implements the FSM interface
* TaskFSM and TaskFSMTracker - an interface and a class that extend FSM and FSMTracker respectively and specialize them for TaskState/TaskStage.
* FsmTaskService (and corresponding FsmTaskFactoryService) - a sample for illustrating usage of TaskFSM and TaskFSMTracker.

Xenon also comes with unit tests for FSMTracker and TaskFSMTracker that help in understanding their usage.

## FSMTracker

To use FSMTracker, you first need to instantiate it and provide it a state machine configuration. The configuration is a map whose keys are the set of states and the value associated with a key is a map itself, holding pairs of transition labels and target states. For example, the following state machine configuration represents a game where the initial state is "STARTED" and the valid transitions from it are "WIN" and "LOSE":

States: "STARTED", "WON", "LOSS"

Transitions:
* "WIN": "STARTED"->"WON"
* "LOSE": "STARTED"->"LOST"

Any transition not explicitly mentioned is considered invalid.

After the state machine is configured, it can be used to:
* Get the current state
* Check if a specific transition is valid, given the current state
* Check if a specific state is a valid target state, given the current state
* Get the set of possible transitions, given the current state
* Get the collection of possible target states, given the current state

Note that for each state, a transition label from that state is unique, however a target state is not necessarily unique because FSM supports multiple transitions from state A to state B (over different 'labels').

## TaskFsmTracker

If the state you're managing with FSM is the built-in TaskState stage, then you don't need to 'build' the FSM configuration yourself: TaskFSMTracker does that for you. The states in TaskFSMTracker are the various TaskState stages (CREATED, STARTED, etc.) and the transition labels are the same (for simplicity). All you need to do is use it, for example (taken from FsmTaskService sample):


    public static class FsmTaskServiceState extends ServiceDocument {
        public TaskFSMTracker fsmInfo;
        public TaskState taskInfo;
    }

    private void validateAndFixInitialState(Operation start) {
        FsmTaskServiceState state = null;

        // client does not have to provide an initial state, but if it provides one it has to be valid
        if (start.hasBody()) {
            state = start.getBody(FsmTaskServiceState.class);
            if (state == null || state.taskInfo == null) {
                throw new IllegalArgumentException(
                        "attempt to initialize service with an empty state");
            }
            if (!TaskStage.CREATED.equals(state.taskInfo.stage)) {
                throw new IllegalArgumentException(String.format(
                        "attempt to initialize service with stage %s != CREATED",
                        state.taskInfo.stage));
            }
        } else {
            state = new FsmTaskServiceState();
            state.taskInfo = new TaskState();
            state.taskInfo.stage = TaskStage.CREATED;
        }

        state.fsmInfo = new TaskFSMTracker();

        start.setBody(state);
    }

    private void validateStateTransitionAndInput(FsmTaskServiceState currentState,
            FsmTaskServiceState newState) {
        if (newState == null || newState.taskInfo == null || newState.taskInfo.stage == null) {
            throw new IllegalArgumentException("new stage is null");
        }

        if (!currentState.fsmInfo.isTransitionValid(newState.taskInfo.stage)) {
            throw new IllegalArgumentException(String.format(
                    "Illegal state transition: current stage=%s, desired stage=%s",
                    currentState.fsmInfo.getCurrentState(),
                    newState.taskInfo.stage));
        }
    }

    private void adjustState(Operation o, FsmTaskServiceState currentState, TaskStage stage) {
        currentState.fsmInfo.adjustState(stage);
        currentState.taskInfo.stage = stage;
        super.setState(o, currentState);
    }
