## State diagram
```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Opening : Command
    Idle --> Closing : !Command
    Opening --> Opened : Feedback
    Opening --> Fault : StateTimer.Q
    Closing --> Closed : !Feedback
    Closing --> Fault : StateTimer.Q
    Opened --> Idle
    Closed --> Idle
    Fault --> Idle : ResetFault
```

## Code
```bash
(* Enum Definition for States *)
TYPE ValveState :
(
    Idle,
    Opening,
    Closing,
    Opened,
    Closed,
    Fault
);
END_TYPE
-------------------------------------------------------------------------------------------------------------------------------------

FUNCTION_BLOCK FB_ValveControl
VAR_INPUT
    Command : BOOL;          // Command to open (TRUE) or close (FALSE) the valve
    Feedback : BOOL;         // Feedback indicating the actual status of the valve
    MaxTime : TIME := T#5S;  // Maximum allowed operation time (default 5 seconds)
    ResetFault : BOOL := FALSE; // Command to reset fault condition
END_VAR

VAR_OUTPUT
    ValveOutput : BOOL;      // Digital output to control the valve
    State : ValveState;      // Current state of the valve
    Fault : BOOL;            // Fault status
END_VAR

VAR
    StateTimer : TON;        // TON timer for operation time monitoring
END_VAR

// State Machine
CASE State OF
    ValveState.Idle:
        IF Command THEN
            State := ValveState.Opening;
        ELSE
            State := ValveState.Closing;
        END_IF
        StateTimer(IN := TRUE, PT := MaxTime);

    ValveState.Opening:
        ValveOutput := TRUE;
        IF Feedback THEN
            State := ValveState.Opened;
            StateTimer(IN := FALSE);
        ELSIF StateTimer.Q THEN
            State := ValveState.Fault;
            StateTimer(IN := FALSE);
        END_IF

    ValveState.Closing:
        ValveOutput := FALSE;
        IF NOT Feedback THEN
            State := ValveState.Closed;
            StateTimer(IN := FALSE);
        ELSIF StateTimer.Q THEN
            State := ValveState.Fault;
            StateTimer(IN := FALSE);
        END_IF

    ValveState.Opened:
		State := ValveState.Idle;
	
    ValveState.Closed:
        State := ValveState.Idle;

    ValveState.Fault:
        Fault := TRUE;
		// Reset Fault and State if ResetFault is TRUE
		IF ResetFault THEN
			Fault := FALSE;
			State := ValveState.Idle;
		END_IF

END_CASE
```
## Usuage
```bash
PROGRAM Main
VAR
    ValveControl : FB_ValveControl;
    OpenCloseSequence : ARRAY[0..6] OF BOOL := [TRUE, FALSE, TRUE, FALSE, TRUE, FALSE, TRUE];
    SequenceIndex : INT := 0;
END_VAR

// Initialize the valve control
ValveControl(MaxTime := T#3M);

// Start the operation sequence
ValveControl(Command := OpenCloseSequence[SequenceIndex]);

// Monitor the valve control state
IF ValveControl.State = ValveState.Opened AND OpenCloseSequence[SequenceIndex] THEN
    // If the valve was supposed to open and it's now open, move to the next operation
    SequenceIndex := SequenceIndex + 1;
    ValveControl(Command := OpenCloseSequence[SequenceIndex]);
ELSIF ValveControl.State = ValveState.Closed AND NOT OpenCloseSequence[SequenceIndex] THEN
    // If the valve was supposed to close and it's now closed, move to the next operation
    SequenceIndex := SequenceIndex + 1;
    ValveControl(Command := OpenCloseSequence[SequenceIndex]);
ELSIF ValveControl.State = ValveState.Fault THEN
    // Handle fault condition
    // ...
END_IF
END_PROGRAM
```
