# ANDROID-BASICS

I. TESTING

https://developer.android.com/training/testing/fundamentals

Dependencies in Gradle are scoped by source set:

- implementation - for the main source set
- testImplementation - for the test source set
- androidTestImplementation - for the androidTest source set

```
// Dependencies for local unit tests
testImplementation "junit:junit:$junitVersion"

// AndroidX Test - Instrumented testing
androidTestImplementation "androidx.test.ext:junit:$androidXTestExtKotlinRunnerVersion"
androidTestImplementation "androidx.test.espresso:espresso-core:$espressoVersion"
```

1. Unit Test

```
class ExampleUnitTest {
    @Test
    fun addition_isCorrect() {
        assertEquals(4, 2 + 2)
    }
}
```

2. Instrumented Test

```
@RunWith(AndroidJUnit4::class)
class ExampleInstrumentedTest {
    @Test
    fun useAppContext() {
        // Context of the app under test.
        val appContext = InstrumentationRegistry.getInstrumentation().targetContext
        assertEquals("com.example.android.architecture.blueprints.reactive",
            appContext.packageName)
    }
}
```

test
androidTest
local test
instrumented test
local machine/jvm
emulator/ real device
faster/ less fidelity
slower/ more fidelity

Readable Tests -

- fn name - subjectUnderTest_actionOrInput_result
- strategy - GIVEN/WHEN/THEN
- Assertion framework - Hamcrest

```
dependencies {
    // Other dependencies
    testImplementation "org.hamcrest:hamcrest-all:$hamcrestVersion"
}
```

```
assertEquals(result.completedTasksPercent, 0f)

// Becomes

assertThat(result.completedTasksPercent, `is`(0f))
```

AutomatedTest

- Unit Test - 70%
- Integration Test - 20%
- E2E Test - 10%

VIEWMODEL TEST

- runs locally - unit test
- if using android viewmodel
  - get application context via androidX libs
    - add androidX dependency
    - add roboelectric dependency

```
    testImplementation "androidx.test.ext:junit:1.1.2"
    testImplementation "androidx.test:core-ktx:1.3.0"
    testImplementation "org.robolectric:robolectric:4.5.1"
```

    	- use androidX test
    	- add androidJUnit test runner

AndroidX test

- collection of libraries for testing
- provides android components for tests
- used in both local and instrumented test

```
@RunWith(AndroidJUnit4::class)
class TasksViewModelTest{

    @Test
    fun addNewTask_newTask(){
        val viewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
        viewModel.addNewTask()
    }
}
```

LIVEDATA TEST

1. use InstantTaskExecuteRule - JUnit rule : classes that allows you to define some code before and after each tests run, ie it runs all arch comprelated background jobs in the same thread. This ensures that the test results happens sync and in repeatable order

```
testImplementation "androidx.arch.core:core-testing:$archTestingVersion"
```

```
class TasksViewModelTest {
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    // Other code…
}
```

2. Observe Livedata via extension fn

```
import androidx.annotation.VisibleForTesting
import androidx.lifecycle.LiveData
import androidx.lifecycle.Observer
import java.util.concurrent.CountDownLatch
import java.util.concurrent.TimeUnit
import java.util.concurrent.TimeoutException


@VisibleForTesting(otherwise = VisibleForTesting.NONE)
fun <T> LiveData<T>.getOrAwaitValue(
    time: Long = 2,
    timeUnit: TimeUnit = TimeUnit.SECONDS,
    afterObserve: () -> Unit = {}
): T {
    var data: T? = null
    val latch = CountDownLatch(1)
    val observer = object : Observer<T> {
        override fun onChanged(o: T?) {
            data = o
            latch.countDown()
            this@getOrAwaitValue.removeObserver(this)
        }
    }
    this.observeForever(observer)

    try {
        afterObserve.invoke()

        // Don't wait indefinitely if the LiveData is not set.
        if (!latch.await(time, timeUnit)) {
            throw TimeoutException("LiveData value was never set.")
        }

    } finally {
        this.removeObserver(observer)
    }

    @Suppress("UNCHECKED_CAST")
    return data as T
}
```

```
@Test
    fun addNewTask_newTask(){
        //Given a new viewmodel
        val viewModel = TasksViewModel(ApplicationProvider.getApplicationContext())

        //when new task is created
        viewModel.addNewTask()
        val value = viewModel.newTaskEvent.getOrAwaitValue()

        //new event is launched
        assertThat(value.getContentIfNotHandled(), (not(nullValue())))
    }
```

Sample test:

```
@RunWith(AndroidJUnit4::class)
class TasksViewModelTest {

    // Subject under test
    private lateinit var tasksViewModel: TasksViewModel

    // Executes each task synchronously using Architecture Components.
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    @Before
    fun setupViewModel() {
        tasksViewModel = TasksViewModel(ApplicationProvider.getApplicationContext())
    }


    @Test
    fun addNewTask_setsNewTaskEvent() {

        // When adding a new task
        tasksViewModel.addNewTask()

        // Then the new task event is triggered
        val value = tasksViewModel.newTaskEvent.awaitNextValue()
        assertThat(
            value?.getContentIfNotHandled(), (not(nullValue()))
        )
    }

    @Test
    fun getTasksAddViewVisible() {

        // When the filter type is ALL_TASKS
        tasksViewModel.setFiltering(TasksFilterType.ALL_TASKS)

        // Then the "Add task" action is visible
        assertThat(tasksViewModel.tasksAddViewVisible.awaitNextValue(), `is`(true))
    }

}
```

REPOSITORY TEST

Test Doubles
Fake - A test double that has a "working" implementation of the class, but it's implemented in a way that makes it good for tests but unsuitable for production

Mock - A test double that tracks which of its methods were called. It then passes or fails a test depending on whether it's methods were called correctly.

Stub - A test double that includes no logic and only returns what you program it to return. A StubTaskRepository could be programmed to return certain combinations of tasks from getTasks for example.

Dummy - A test double that is passed around but not used, such as if you just need to provide it as a parameter. If you had a DummyTaskRepository, it would just implement the TaskRepository with no code in any of the methods.

Spy - A test double which also keeps tracks of some additional information; for example, if you made a SpyTaskRepository, it might keep track of the number of times the addTask method was called.
