# JSON Schema for OpenAI API

ref: https://platform.openai.com/docs/guides/structured-outputs?api-mode=chat#supported-schemas


## Python Code Generation
After generating the JSON schema, you can use it to generate Python classes using Pydantic. Here's how you can do it:

```
pip install datamodel-code-generator
# save the JSON schema to a file named json_schema.json
datamodel-codegen --input json_schema.json --input-file-type jsonschema --output model.py
```

reference: https://docs.pydantic.dev/latest/integrations/datamodel_code_generator/
