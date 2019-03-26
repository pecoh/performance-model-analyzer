Basic format: JSON

Top level is an object with members:

  * resources: dictionary of objects with members:

      * throughput: number (always units per second)

        not necessary if multiplicity is given

      * multiplicity: number (integer giving max number of concurrent jobs using this resource)

        default: inf

      * resources: recursive dictionary

        default: empty

  * job definitions: dictionary of objects with members:

      * resources: array of objects with members:

          * path: array of strings (which form a path into the resources tree)

          * require: number (amount of the resource that is consumed)

  * jobs: dictionary of objects with members:

      * name: string (name of a job definition)

      * multiplicity: number (integer giving the number of identical jobs that are created)

        default: 1

        This duplicates the work (weak scaling).

      * split factor: number (integer giving the number of identical jobs that are created)

        default: 1

        This splits the work across the given number of identical jobs (strong scaling).
        "multiplicity" and "split factor" can be combined, so if "multiplicity" == 3 and "split factor" == 4,
        then 12 jobs are created, each with a quarter of the resource requirements given in the job definition,
        so that three times the resources given in the job definition are required in total.

      * require: array of strings (the names of jobs that need to finish before this job can be started.

        default: empty

        prerequisites with a multiplicity are only considered to be fulfilled when all the duplicate jobs have finished.

Top level object may be split into several objects which are provided by different files.

Example data:

{
	"resources": {
		"CPU": {
			"multiplicity": 4,
			"resources": {
				"core": {
					"multiplicity": 2,
					"resources": {
						"thread": {
							"multiplicity": 1,
							"resources": {
								"processing": { "throughput": 3e9 }
							}
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

	"job definitions": {
		"read config": {
			"resources": [
				{ "path": ["network"], "require": 50e3},
				{ "path": ["CPU", "memory bus"], "require": 50e3},
				{ "path": ["CPU", "LL Cache"], "require": 50e3},
				{ "path": ["CPU", "core", "L1 Cache"], "require": 500e3},
				{ "path": ["CPU", "core", "thread", "processing"], "require": 1e6}
			]
		},
		"load input": {
			"resources": [
				{ "path": ["network"], "require": 50e9},
				{ "path": ["CPU", "memory bus"], "require": 50e9},
				{ "path": ["CPU", "LL Cache"], "require": 150e9},
				{ "path": ["CPU", "core", "L1 Cache"], "require": 150e9},
				{ "path": ["CPU", "core", "thread", "processing"], "require": 100e9}
			]
		},
		"process data": {
			"resources": [
				{ "path": ["CPU", "memory bus"], "require": 150e9},
				{ "path": ["CPU", "LL Cache"], "require": 300e9},
				{ "path": ["CPU", "core", "L1 Cache"], "require": 300e9},
				{ "path": ["CPU", "core", "thread", "processing"], "require": 1e12}
			]
		}
	},

	"jobs": {
		"read config": { "name": "read config", "multiplicity": 4 },
		"load input": { "name": "load input", "require": [ "read config" ]}
		"process data": { "name": "process data", "split factor": 4, "require": [ "load input" ]},
	}
}
