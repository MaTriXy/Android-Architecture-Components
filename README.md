# Android Architecture Components
![](https://cdn-images-1.medium.com/max/800/1*WVdFMYmEoCdXniy7ulDe5g.png)


## Read article [here](https://medium.com/proandroiddev/android-architecture-components-cb1ea88d3835)
Android Architecture Components (AAC) is a new collection of libraries that contains the lifecycle-aware components. It can solve problems with configuration changes, supports data persistence, reduces boilerplate code, helps to prevent memory leaks and simplifies async data loading into your UI. I can’t say that it brings absolutely new approaches for solving these issues, but, finally, we have a formal, single and official direction.

AAC provides some abstractions to deal with Android lifecycle:
* LifecycleOwner
* LiveData
* ViewModel

The main benefit is the fact that our UI components, like TextView or RecycleView, observe LiveData, which, in turn, observes the lifecycle of an Activity or Fragment, using a LifecycleObserver.

![](https://cdn-images-1.medium.com/max/800/1*KD4TON1jgYjnc7CQuxJIrA.png)

Combination of these components solves main challenges faced by Android developers, such as boilerplate code or modular. To explore and check an example of this concept, I decided to create the sample project. It just gets a list of repositories from Github and shows one using RecyclerView.

![](https://cdn-images-1.medium.com/max/800/1*YTzzwEjS0NNnrxmlgXr6ZA.gif)

As you can see, it handles configuration changes without any problems, and an Activity looks very simple:

```Java
class ReposActivity : BaseLifecycleActivity<ReposViewModel>(), SwipeRefreshLayout.OnRefreshListener {

    override val viewModelClass = ReposViewModel::class.java

    private val rv by unsafeLazy { findViewById<RecyclerView>(R.id.rv) }

    private val vRefresh by unsafeLazy { findViewById<SwipeRefreshLayout>(R.id.lRefresh) }

    private val adapter = ReposAdapter()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_repos)
        rv.setHasFixedSize(true)
        rv.adapter = adapter
        vRefresh.setOnRefreshListener(this)

        if (savedInstanceState == null) {
            viewModel.setOrganization("yalantis")
        }
        observeLiveData()
    }

    private fun observeLiveData() {
        viewModel.isLoadingLiveData.observe(this, Observer<Boolean> {
            it?.let { vRefresh.isRefreshing = it }
        })
        viewModel.reposLiveData.observe(this, Observer<List<Repo>> {
            it?.let { adapter.dataSource = it }
        })
        viewModel.throwableLiveData.observe(this, Observer<Throwable> {
            it?.let { Snackbar.make(rv, it.localizedMessage, Snackbar.LENGTH_LONG).show() }
        })
    }

    override fun onRefresh() {
        viewModel.setOrganization("yalantis")
    }
}
```

How you have probably noticed, our activity assumes minimum responsibilities. ReposViewModel holds state and view data in the following way:

```Java
open class ReposViewModel(application: Application?) : AndroidViewModel(application) {

    private val organizationLiveData = MutableLiveData<String>()

    val resultLiveData = ReposLiveData().apply {
        this.addSource(organizationLiveData) { it?.let { this.organization = it } }
    }

    val isLoadingLiveData = MediatorLiveData<Boolean>().apply {
        this.addSource(resultLiveData) { this.value = false }
    }

    val throwableLiveData = MediatorLiveData<Throwable>().apply {
        this.addSource(resultLiveData) { it?.second?.let { this.value = it } }
    }

    val reposLiveData = MediatorLiveData<List<Repo>>().apply {
        this.addSource(resultLiveData) { it?.first?.let { this.value = it } }
    }

    fun setOrganization(organization: String) {
        organizationLiveData.value = organization
        isLoadingLiveData.value = true
    }

}
```

Testability

```Java
@RunWith(AndroidJUnit4::class)
class SampleInstrumentedTest {

    @get:Rule
    val activityRule = ActivityTestRule<ReposActivity>(ReposActivity::class.java, true, true)

    private var viewModel: ReposViewModel? = null

    @Before
    fun init() {
        viewModel = ViewModelProviders.of(activityRule.activity).get(ReposViewModel::class.java)
    }

    @Test
    fun testNotNull() {
        activityRule.activity.runOnUiThread {
            viewModel?.setOrganization("yalantis")
            viewModel?.reposLiveData?.observe(activityRule.activity, Observer<List<Repo>> {
               assertNotNull(it)
            })
        }
    }
}
```

