# Industrial Automation - Engineering Log 
**Date:** Jan 2026  
**Hardware:** ABB IRB 1200 (Virtual Controller)  
**Driver:** abb-motion-program-exec  

## Goal
Bypass the standard Teach Pendant programming to control the robot via an external Python script. This is to establish a workflow for "Sim-to-Real" deployment, where trajectories generated in simulation (Isaac Sim) can be executed on hardware.

## Technical Notes

### 1. Work Objects (wobj) vs. Base Frame
Initially confused why we couldn't just plan relative to the robot base.
* **Issue:** If the fixture/table moves relative to the robot, hard-coded coordinates become useless.
* **Solution:** Defined a `wobj` at `[600, 0, 550]`. This creates a local coordinate system. Moving the table only requires updating this one definition rather than re-teaching all points.

### 2. Zoning Strategies
* `fine`: Robot comes to a complete stop (velocity = 0). Essential for the start/end of weld seams.
* `z50`: Robot maintains velocity but blends the corner by 50mm. Much smoother for air moves.

### 3. Controller Configuration (PC Interface)
Python script initially failed to connect. Found that the standard virtual controller doesn't expose the RWS ports needed for external control.  
* **Fix:** Rebuilt the system in RobotStudio with option **616-1 PC Interface** enabled. This opened the socket for the Python driver.

### 4. Data Verification
* **Objective:** Ensure the robot actually reached the commanded targets within the `fine` zone tolerance.
* **Method:** Plotted the feedback signal (`joint_states`) returned by the RWS client.
* **Result:** Confirmed smooth interpolation between waypoints with no unexpected stops or jerky motion (discontinuities in velocity).

## Debugging
### Network Configuration (Sim vs. Real)
The Python driver failed to connect initially because the IP was hardcoded to a static industrial address (`192.168.125.1`), which is standard for the physical IRC5 controller.
* **Fix:** Switched to `127.0.0.1` (localhost) for the RobotStudio simulation.
* **Architecture Note:** In production, this should not be hardcoded. It needs to be an environment variable or config parameter to allow seamless Sim-to-Real deployment without code changes.

### File Path Error
Ran into a `FileNotFoundError` when executing the driver from VS Code. The script was looking for the config YAML in the current working directory rather than relative to the script location.
* **Fix:** Standardized path finding using `os`.

```python
# Old (Failed)
# with open('config.yml', 'r') as file:

# New (Fixed)
script_dir = os.path.dirname(os.path.abspath(__file__))
config_path = os.path.join(script_dir, '..', 'config', 'config.yml')
with open(config_path, 'r') as file:
```
## Safety & ISO 10218 Compliance

### Hardware vs. Software Hierarchy
* **Observation:** Attempted to write a Python `try/except` block to auto-recover the robot after an E-Stop trigger.
* **Failure:** The robot controller rejected all commands.
* **Key Learning:** Safety functions are hierarchically superior to software control. The IRC5 controller enters a **latching hardware state** (Category 0 Stop) that physically disconnects motor power. No software command can override this.

### Recovery Sequence (Deadman Logic)
* **Standard:** ISO 10218-1 requires specific enabling device logic for manual mode.
* **Workflow:** To resume Python control after a violation, a human must physically interact with the cell:
    1.  **Reset:** Twist to release the E-Stop button.
    2.  **Acknowledge:** Clear the error on the FlexPendant UI.
    3.  **Enable:** Press the "Motor On" (White Button) hardware switch.
    4.  **Engage:** Lightly press the Deadman switch to the center position (Position 2).
