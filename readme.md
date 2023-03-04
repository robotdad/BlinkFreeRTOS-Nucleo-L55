# Nucleo-L552ZE-Q

This repository was created by following the steps below. This walks you through creating a FreeRTOS project in CubeMX, importing it into VS Code, building it, adding application code to blink LEDs on the device, and debugging the running firmware on the board.

To use this just clone this repo and you can open in VS Code. Make sure you have completed the setup steps, then you can jump ahead to building the firmware.

```
git clone https://github.com/robotdad/BlinkFreeRTOS-Nucleo-L55.git
cd BlinkFreeRTOS-Nucleo-L55
code .
```
# Setup
Have CubeMX, CubeIDE, and VS Code with Embedded Tools VS Code Extension installed.

Make sure you have updated your STLink firmware on your board.

I have included the steps for creating git repos and connecting them to GitHub below. If you are following along with those steps as well you need git and the GitHub cli installed.

# FreeRTOS from scratch with CubeMX

In CubeMX go to board selector, search for the Nucleo-L552ZE-Q, select it, and start project. Initialize all peripherals to default state, do not enable TrustZone.

Starting on the Pinout tab, under Categories, Middleware select FreeRTOS and enable CMSIS_V2. Select Tasks and Queues under the Configuration options. Change the default task name to blinkRed and the entry function to StartBlinkRed. Add a new task blinkGreen, entry function StartBlinkGreen, and set osPriorityBelowNormal. Add a new task blinkBlue, entry function StartBlinkBlue, and set osPriorityLow. Select Advanced settings and anable USE_NEWLIB_REENTRANT.

Under Categories, System Code, select Sys and change the Time base source to TIM6. Now select ICACHE and enable 1-way mode.

Select the Project Manager tab give it a name and select your project location. Select STM32CubeIDE for the toolchain and generate under root.

Navigate to where you saved your generated project and setup a git repo.
```
git init -b main
git add . && git commit -m "initial commit"
gh repo create
```
Follow the prompts for creating the repo on GitHub. You will want to add a remote, defaults are fine.

## Importing an ST Project
Open VS Code, in the command palette (Ctrl + Shift + P) use Create project from ST Project. This will launch the file browser, navigate to where the project you created in CubeMX is, and select the .cproject. If prompted to trust the authors, agree. You will be prompted to choose a configuration, select debug.

The result of the above actions is that a CMake project will be generated, default tasks will be created for the project for use in VS Code and VS, and a vcpkg-configuration file will be created and activated. The vcpkg step configures the environment so that CMake, Ninja, and the ST compilers from the STM32IDE are all on path and ready to use.

In VS Code you can select the git activity and commit the generated artifacts from the import.

You should now create a .gitignore file. Minimally make sure to exclude the build folder by adding the line "build/" to the file. You can also copy [a complete .gitignore from here](https://raw.githubusercontent.com/robotdad/BlinkFreeRTOS-Nucleo-L55/main/.gitignore). When done commit the file. 

## Updating tasks
We want to add implementations to the blink tasks we generated in CubeMX. In the file explorer in VS Code expand the folder Core > Src and select main.c. Select Ctrl + T to search for symbols, and search for StartBlinkRed and select it (not the declaration).

Notice that there are comments of USER CODE BEGIN and USER CODE END throughout the source. Always place your changes between these blocks as it will be preserved if you need to regenerate code using CubeMX. You may need to do that if you make changes to peripheral options under pinout for example.

In the user code for StartBlinkRed there is an infinite loop with an osDelay call. This function is called by FreeRTOS as a task so it runs in parallel to other tasks. Delete the osDelayCall and add the following lines.
```
HAL_GPIO_TogglePin(LED_RED_GPIO_Port, LED_RED_Pin);
osDelay(1000);
```
Now, to understand what is being used on the hardware right select LED_RED_GPIO_Port and select Peek, Peek Definition. You will see that this is defined as GPIOA. Do the same thing for LED_RED_PIN, or select Alt F12, and notice it is GPIO Pin 9.

Let's complete the implementation for StartBlinkGreen as above.
```
HAL_GPIO_TogglePin(LED_GREEN_GPIO_Port, LED_GREEN_Pin);
osDelay(1000);
```
Using peek definition again note this corresponds to GPIO C, Pin 7.

Now for StartBlinkBlue.
```
HAL_GPIO_TogglePin(LED_BLUE_GPIO_Port, LED_BLUE_Pin);
osDelay(1000);
```
Using peek definition again note this corresponds to GPIO B, Pin 7.

## Building the firmware

Select the CMakeLists.txt file in the file explorer and from the context menu select Clean Reconfigure All Projects. This ensures we have generated the build files using the proper compilers that are now on the path. From the context menu for CMakeLists.txt you can now select Build All Projects. This should build your firmware under the build directory.

## Debugging
Select the Debug icon in the activity bar. Launch should already be selected. You can select the gear icon to see the details of the provided launch configuration. Note that the miDebuggerPath and debugServerPath are configured using environment variables provided by the Embedded Tools extension that use ST's gdb and gdbserver executables provided by STM32IDE.

The postRemoteConnectCommands has been generated with the firmware name from the CMake project target to be loaded onto the device. There is also an svdPath configured to load the correct SVD file for our part.

By default stopAtConnect is set to True which will cause us to break in the startup handler.

Place a breakpoint in any of the StartBlink functions created above.

Connect your board and select the play button to start debugging, or press F5.

When you hit a break point open the peripheral view from the command palette, select Focus on peripheral view. Expand the view that opens in the debug pane. Find the GPIO section corresponding to your breakpoint. The pins are under the ODR section. You can step over the HAL_GPIO_TogglePin call and either watch the value for ODR change, or expand it to see the specific pin value update.

GPIOs for LEDs:
* Red: GPIOA ODR9
* Green: GPIOC ODR7
* Blue: GPIOB ODR7