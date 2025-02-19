.. _generator:

Generator
=========

.. .. admonition:: Author
..    :class: highlight

..    `Li Yin <https://github.com/liyin2015>`_

.. *The Center of it All*

`Generator` is a user-facing orchestration component with a simple and unified interface for LLM prediction.
It is a pipeline that consists of three subcomponents.

Design
---------------------------------------

.. figure:: /_static/images/generator.png
    :align: center
    :alt: LightRAG generator design
    :width: 700px

    Generator-The orchestrator for LLM prediction

:class:`Generator<core.generator.Generator>` is designed to achieve the following goals:

- Model Agnostic: The Generator should be able to call any LLM model with the same prompt.
- Unified Interface: It should manage the pipeline of prompt(input)->model call -> output parsing.
- Unified Output: This will make it easy to log and save records of all LLM predictions.
- Work with Optimizer: It should be able to work with Optimizer to optimize the prompt.

An orchestrator
^^^^^^^^^^^^^^^^^

It orchestrates three components:

- ``Prompt``: by taking in ``template`` (string) and ``prompt_kwargs`` (dict) to format the prompt at initialization. When the ``template`` is not given, it defaults to :const:`DEFAULT_LIGHTRAG_SYSTEM_PROMPT<core.default_prompt_template.DEFAULT_LIGHTRAG_SYSTEM_PROMPT>`.

- ``ModelClient``: by taking in already instantiated ``model_client`` and ``model_kwargs`` to call the model. Switching out the model client will allow you to call any LLM model on the same prompt and output parsing.

- ``output_processors``: component or chained components via ``Sequential`` to process the raw response to desired format. If no output processor provided, it is decided by Model client, often return raw string response (from the first response message).

**Call and arguments**

Generator supports both ``call`` (``__call__``) and ``acall`` method.
They take two optional arguments:

- ``prompt_kwargs`` (dict): to be passed to its ``Prompt`` component.
- ``model_kwargs`` (dict): will be combined with the ``model_kwargs`` from the initial model client.

The generator will call ``Prompt`` to format the final prompt and adapt the inputs to ones workable with ``ModelClient``.
In particular, it passes:

- Formatted prompt after calling ``Prompt``.
- All combined ``model_kwargs`` and :const:`ModelType.LLM<core.types.ModelType.LLM>` to the ``ModelClient``.

.. note ::

    This also means any ``ModelClient`` who wants to be compatible with `Generator` should take in ``model_kwargs`` and ``model_type`` as arguments.



GeneratorOutput
^^^^^^^^^^^^^^^^^
Different from all other components, we can not alway enforce LLM to output the right format.
Some part of the `Generator` pipeline can fail.

.. note::
    Whenever there is an error happens, we do not raise the error and stop this pipeline.
    Instead, `Generator` will always return an output record.
    We made this design choice as it can be really helpful to log various failed cases on your test examples without stopping the pipeline for further investigation and improvement.

In particular, we created :class:`GeneratorOutput<core.types.GeneratorOutput>` (subclass of ``DataClass``) to capture important information.

- `data` (object) : to store the final processed response after all three components in the pipeline. This means `success`.
- `error` (str): error message if any of the three components in the pipeline fail. When this is not `None`, it means `failure`.
- `raw_response` (str): raw string response for reference for any LLM predictions. For now it is a string, which comes from the first response message. [This might change and be different in the future]
- `metadata` (dict): to store any additional information and `usage` reserved to track the usage of the LLM prediction.

Whether to do further processing or terminate the pipeline whenever an error happens is up to the user from here on.

Generator In Action
---------------------------------------

We will create a simple one-turn chatbot to demonstrate how to use the Generator in action.

Minimum Example
^^^^^^^^^^^^^^^^^

The minimum setup to initiate a generator in the code:

.. code-block:: python

    from lightrag.core import Generator
    from lightrag.components.model_client import GroqAPIClient

    generator = Generator(
        model_client=GroqAPIClient(),
        model_kwargs={"model": "llama3-8b-8192"},
    )
    print(generator)

The structure of generator using ``print``:

.. raw:: html

    <div style="max-height: 300px; overflow-y: auto;">
        <pre>
            <code class="language-python">
        Generator(
        model_kwargs={'model': 'llama3-8b-8192'},
        (prompt): Prompt(
            template: {% if task_desc_str or output_format_str or tools_str or examples_str or chat_history_str or context_str or steps_str %}
            <SYS>
            {% endif %}
            {# task desc #}
            {% if task_desc_str %}
            {{task_desc_str}}
            {% endif %}
            {# output format #}
            {% if output_format_str %}
            <OUTPUT_FORMAT>
            {{output_format_str}}
            </OUTPUT_FORMAT>
            {% endif %}
            {# tools #}
            {% if tools_str %}
            <TOOLS>
            {{tools_str}}
            </TOOLS>
            {% endif %}
            {# example #}
            {% if examples_str %}
            <EXAMPLES>
            {{examples_str}}
            </EXAMPLES>
            {% endif %}
            {# chat history #}
            {% if chat_history_str %}
            <CHAT_HISTORY>
            {{chat_history_str}}
            </CHAT_HISTORY>
            {% endif %}
            {#contex#}
            {% if context_str %}
            <CONTEXT>
            {{context_str}}
            </CONTEXT>
            {% endif %}
            {# steps #}
            {% if steps_str %}
            <STEPS>
            {{steps_str}}
            </STEPS>
            {% endif %}
            {% if task_desc_str or output_format_str or tools_str or examples_str or chat_history_str or context_str or steps_str %}
            </SYS>
            {% endif %}
            {% if input_str %}
            <User>
            {{input_str}}
            </User>
            {% endif %}
            You:
            , prompt_variables: ['output_format_str', 'chat_history_str', 'task_desc_str', 'context_str', 'steps_str', 'input_str', 'tools_str', 'examples_str']
        )
        (model_client): GroqAPIClient()
    )
            </code>
        </pre>
    </div>

**Show the final prompt**

`Generator` 's ``print_prompt`` method will simply relay the method from the `Prompt` component:

.. code-block:: python

    prompt_kwargs = {"input_str": "What is LLM? Explain in one sentence."}
    generator.print_prompt(**prompt_kwargs)

The output will be the formatted prompt:

.. code-block::

    <User>
    What is LLM? Explain in one sentence.
    </User>
    You:



**Call the generator**

.. code-block:: python

    output = generator(
        prompt_kwargs=prompt_kwargs,
    )
    print(output)

The output will be the `GeneratorOutput` object:

.. code-block::

    GeneratorOutput(data='LLM stands for Large Language Model, a type of artificial intelligence that is trained on vast amounts of text data to generate human-like language outputs, such as conversations, text, or summaries.', error=None, usage=None, raw_response='LLM stands for Large Language Model, a type of artificial intelligence that is trained on vast amounts of text data to generate human-like language outputs, such as conversations, text, or summaries.', metadata=None)

Use template
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In this example, we will use a customized template to format the prompt.
We intialized the prompt with one variable `task_desc_str` and it is further combined with the `input_str` in the prompt.

.. code-block:: python

    template = r"""<SYS>{{task_desc_str}}</SYS>
    User: {{input_str}}
    You:"""
    generator = Generator(
        model_client=GroqAPIClient(),
        model_kwargs={"model": "llama3-8b-8192"},
        template=template,
        prompt_kwargs={"task_desc_str": "You are a helpful assistant"},
    )

    prompt_kwargs = {"input_str": "What is LLM?"}

    generator.print_prompt(
        **prompt_kwargs,
    )
    output = generator(
        prompt_kwargs=prompt_kwargs,
    )

The final prompt is:

.. code-block::

    <SYS>You are a helpful assistant</SYS>
    User: What is LLM?
    You:

.. note::

    It is quite straightforward to use any prompt.
    They only need to stick to ``jinja2`` syntax.


Use output_processors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In this example, we will instruct LLM to output a JSON object to respond.
We will use the `JsonParser` to parse the output back to a `dict` object.

.. code-block:: python

    from lightrag.core import Generator
    from lightrag.core.types import GeneratorOutput
    from lightrag.components.model_client import OpenAIClient
    from lightrag.core.string_parser import JsonParser

    output_format_str = r"""Your output should be formatted as a standard JSON object with two keys:
    {
        "explaination": "A brief explaination of the concept in one sentence.",
        "example": "An example of the concept in a sentence."
    }
    """

    generator = Generator(
        model_client=OpenAIClient(),
        model_kwargs={"model": "gpt-3.5-turbo"},
        prompt_kwargs={"output_format_str": output_format_str},
        output_processors=JsonParser(),
    )

    prompt_kwargs = {"input_str": "What is LLM?"}
    generator.print_prompt(**prompt_kwargs)

    output: GeneratorOutput = generator(prompt_kwargs=prompt_kwargs)
    print(type(output.data))
    print(output.data)

The final prompt is:

.. code-block::


    <SYS>
    <OUTPUT_FORMAT>
    Your output should be formatted as a standard JSON object with two keys:
        {
            "explaination": "A brief explaination of the concept in one sentence.",
            "example": "An example of the concept in a sentence."
        }

    </OUTPUT_FORMAT>
    </SYS>
    <User>
    What is LLM?
    </User>
    You:

The output of the call is:

.. code-block::

    <class 'dict'>
    {'explaination': 'LLM stands for Large Language Model, which are deep learning models trained on enormous amounts of text data.', 'example': 'An example of a LLM is GPT-3, which can generate human-like text based on the input provided.'}

Switch model client
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Also, did you notice that we have already switched to use models from `OpenAI` in the above example?
This is how easy to switch the model client in the Generator, making it a truly model-agnostic component.
We can even use :class:`ModelClientType<core.types.ModelClientType>` to switch the model client without handling multiple imports.

.. code-block:: python

    from lightrag.core.types import ModelClientType

    generator = Generator(
        model_client=ModelClientType.OPENAI(),  # or ModelClientType.GROQ()
        model_kwargs={"model": "gpt-3.5-turbo"},
    )

Get errors in the output
^^^^^^^^^^^^^^^^^^^^^^^^^

We will use a wrong API key to delibrately create an error.
We will still get a response, but only with empty ``data`` and an error message.
Here is the api key error with OpenAI:

.. code-block:: python

    GeneratorOutput(data=None, error="Error code: 401 - {'error': {'message': 'Incorrect API key provided: ab. You can find your API key at https://platform.openai.com/account/api-keys.', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_api_key'}}", usage=None, raw_response=None, metadata=None)


Create from configs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Same as all components, we can create the generator purely from configs.

**Know it is a Generator**

In this case, we know we are creating a generator, we will use ``from_config`` method from the ``Generator`` class.

.. code-block:: python

    from lightrag.core import Generator

    config = {
        "model_client": {
            "component_name": "GroqAPIClient",
            "component_config": {},
        },
        "model_kwargs": {
            "model": "llama3-8b-8192",
        },
    }

    generator: Generator = Generator.from_config(config)
    print(generator)

    prompt_kwargs = {"input_str": "What is LLM? Explain in one sentence."}
    generator.print_prompt(**prompt_kwargs)
    output = generator(
        prompt_kwargs=prompt_kwargs,
    )
    print(output)


**Purely from the configs**

This is even more general.
This method fits to create any component from configs.
We just need to follow the config structure: ``component_name`` and ``component_config`` for all arguments.

.. code-block:: python

    from lightrag.utils.config import new_component
    from lightrag.core import Generator

    config = {
        "generator": {
            "component_name": "Generator",
            "component_config": {
                "model_client": {
                    "component_name": "GroqAPIClient",
                    "component_config": {},
                },
                "model_kwargs": {
                    "model": "llama3-8b-8192",
                },
            },
        }
    }

    generator: Generator = new_component(config["generator"])
    print(generator)

    prompt_kwargs = {"input_str": "What is LLM? Explain in one sentence."}
    generator.print_prompt(**prompt_kwargs)
    output = generator(
        prompt_kwargs=prompt_kwargs,
    )
    print(output)

It works exactly the same as the previous example.
We imported ``Generator`` in this case to only show the type hinting.

.. note::

    Please refer the :doc:`configurations<configs>` for more details on how to create components from configs.


Examples across the library
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Beside of these examples, LLM is like water, even in our library, we have components that have adpated Generator to other various functionalities.

- :class:`LLMRetriever<components.retriever.llm_retriever.LLMRetriever>` is a retriever that uses Generator to call LLM to retrieve the most relevant documents.
- :class:`DefaultLLMJudge<eval.llm_as_judge.DefaultLLMJudge>` is a judge that uses Generator to call LLM to evaluate the quality of the response.
- :class:`LLMOptimizer<optim.llm_optimizer.LLMOptimizer>` is an optimizer that uses Generator to call LLM to optimize the prompt.

Tracing
---------------------------------------
In particular, we provide two tracing methods to help you develop and improve the ``Generator``:

1. Trace the history change (states) on prompt during your development process.

Developers typically go through a long process of prompt optimization, and it is frustrating to lose track of the prompt changes when your current change actually makes the performance much worse.
We created a :class:`GeneratorStateLogger<tracing.generator_state_logger.GeneratorStateLogger>` to handle the logging and saving into JSON files. To further simplify the developer's process, we provide a class decorator `trace_generator_states` where a single line of code can be added to any of your task components. It will automatically track any attributes of type `Generator`.

2. Trace all failed LLM predictions for further improvement.

Similarly, :class:`GeneratorCallLogger<tracing.generator_call_logger.GeneratorCallLogger>` is created to log generator call input arguments and output results.
The `trace_generator_call` decorator is provided to offer a one-line setup to trace calls, which by default will log only failed predictions.

.. note::

    This note is getting rather long. Please go to the :doc:`tracing<logging_tracing>` for more details on how to use these tracing methods.



Training [Experimental]
---------------------------------------
Coming soon!

.. A Note on Tokenization#
.. By default, LlamaIndex uses a global tokenizer for all token counting. This defaults to cl100k from tiktoken, which is the tokenizer to match the default LLM gpt-3.5-turbo.

.. If you change the LLM, you may need to update this tokenizer to ensure accurate token counts, chunking, and prompting.

.. admonition:: API reference
   :class: highlight

   - :class:`core.generator.Generator`
   - :class:`core.types.GeneratorOutput`
   - :class:`core.default_prompt_template.DEFAULT_LIGHTRAG_SYSTEM_PROMPT`
   - :class:`core.types.ModelClientType`
   - :class:`core.types.ModelType`
   - :class:`core.string_parser.JsonParser`
   - :class:`core.prompt_builder.Prompt`
   - :class:`tracing.generator_call_logger.GeneratorCallLogger`
   - :class:`tracing.generator_state_logger.GeneratorStateLogger`
   - :class:`components.retriever.llm_retriever.LLMRetriever`
   - :class:`eval.llm_as_judge.DefaultLLMJudge`
   - :class:`optim.llm_optimizer.LLMOptimizer`
   - :func:`utils.config.new_component`
