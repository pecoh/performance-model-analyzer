Basic format: JSON

Top level is an object with members:

  * resources: dictionary of objects with members:

      * throughput: number (always units per second)

        not necessary if recursive resources are present

      * timeshared: bool (if the resoure is used by several jobs, each job can use only a fraction of the (sub-)resources that corresponds to its time share)

        A job can only use a total of one of each timeshared resource type,
        i.e. it may only use a total of one CPU thread, which may be allocated to different cores,
        but it may additionally use one GPU on top of its CPU usage because the GPU is a different resource type.

        default: false

      * multiplicity: number (this says how many times the resource is available)

        default: 1

      * resources: recursive dictionary

        default: empty

  * operations: dictionary of objects with members:

      * resources: array of objects with members:

          * path: array of strings (which form a path into the resources tree)

          * require: number (amount of the resource that is consumed)

  * jobs: dictionary of objects with members:

      * name: string (name of an operation)

      * multiplicity: number (integer giving the number of identical jobs that are created)

        **unimplemented**

        default: 1

        This duplicates the work (weak scaling).

      * split factor: number (integer giving the number of identical jobs that are created)

        **unimplemented**

        default: 1

        This splits the work across the given number of identical jobs (strong scaling).
        "multiplicity" and "split factor" can be combined, so if "multiplicity" == 3 and "split factor" == 4,
        then 12 jobs are created, each with a quarter of the resource requirements given in the operation,
        so that three times the resources given in the operation are required in total.

      * require: array of strings (the names of jobs that need to finish before this job can be started.

        default: empty

        prerequisites with a multiplicity are only considered to be fulfilled when all the duplicate jobs have finished.

      * pipe: dictionary with members:

        **unimplemented**

          * input: string (name of job from which data is piped)

          * wait factor: number (in the range [0, 1] giving the amount that the input job needs to be ahead of the dependent job)

            default: 0

            A wait factor of 1 is equivalent to mentioning the input job in the "require" attribute, the input job needs to finish before the dependent job can start.
            A value of 0 means that there is no communication ovehead (which is impossible, of course), and that the dependent job may never overtake the input job.
            A value of 0.1 would mean that when the input job is 35% complete, the dependent job cannot be more than 25% complete.

Top level object may be split into several objects which are provided by different files.

Example data:

{
	"resources": {
		"CPU": {
			"multiplicity": 2,
			"resources": {
				"core": {
					"multiplicity": 8,
					"resources": {
						"thread": {
							"multiplicity": 2,
							"timeshared": true,
							"throughput": 3e9
						},
						"L1 Cache": { "throughput": 10e9 }
					}
				},
				"LL Cache": { "throughput": 15e9 },
				"memory bus": { "throughput": 10e9 }
			}
		},
		"network": { "throughput": 2.5e9 }
	},

	"operations": {
		"read config": {
			"resources": [
				{ "path": ["network"], "require": 50e3},
				{ "path": ["CPU", "memory bus"], "require": 50e3},
				{ "path": ["CPU", "LL Cache"], "require": 50e3},
				{ "path": ["CPU", "core", "L1 Cache"], "require": 500e3},
				{ "path": ["CPU", "core", "thread"], "require": 1e6}
			]
		},
		"load input": {
			"resources": [
				{ "path": ["network"], "require": 50e9},
				{ "path": ["CPU", "memory bus"], "require": 50e9},
				{ "path": ["CPU", "LL Cache"], "require": 150e9},
				{ "path": ["CPU", "core", "L1 Cache"], "require": 150e9},
				{ "path": ["CPU", "core", "thread"], "require": 100e9}
			]
		},
		"process data": {
			"resources": [
				{ "path": ["CPU", "memory bus"], "require": 150e9},
				{ "path": ["CPU", "LL Cache"], "require": 300e9},
				{ "path": ["CPU", "core", "L1 Cache"], "require": 300e9},
				{ "path": ["CPU", "core", "thread"], "require": 1e12}
			]
		}
	},

	"jobs": {
		"read config": { "name": "read config", "multiplicity": 4 },
		"load input": { "name": "load input", "require": [ "read config" ]}
		"process data": { "name": "process data", "split factor": 4, "pipe": { "input": "load input", "wait factor": 0.1}},
	}
}
