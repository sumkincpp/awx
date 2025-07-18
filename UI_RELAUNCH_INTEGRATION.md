# Job Relaunch UI Integration Guide

This document provides guidance for implementing UI support for the enhanced job relaunch functionality in the ansible-ui repository.

## Backend API Changes (Complete)

The AWX backend now supports enhanced job relaunch functionality that allows modifying promptable fields during relaunch. The API endpoint is:

```
POST /api/v2/jobs/{id}/relaunch/
```

## Supported Promptable Fields

The following fields can be modified during relaunch if they are marked as promptable in the job template:

| Field | Job Template Setting | Type | Description |
|-------|---------------------|------|-------------|
| `inventory` | `ask_inventory_on_launch` | integer | Inventory ID to run the job against |
| `limit` | `ask_limit_on_launch` | string | Further limit selected hosts to an additional pattern |
| `scm_branch` | `ask_scm_branch_on_launch` | string | Branch to use from source control |
| `job_tags` | `ask_tags_on_launch` | string | Playbook tags to apply |
| `skip_tags` | `ask_skip_tags_on_launch` | string | Playbook tags to skip |
| `extra_vars` | `ask_variables_on_launch` | object | YAML or JSON formatted extra variables |
| `verbosity` | `ask_verbosity_on_launch` | integer | Verbosity level for the job (0-5) |
| `diff_mode` | `ask_diff_mode_on_launch` | boolean | Enable diff mode for the job |
| `forks` | `ask_forks_on_launch` | integer | Number of parallel processes to use |
| `job_slice_count` | `ask_job_slice_count_on_launch` | integer | Number of slices to split the job into |
| `timeout` | `ask_timeout_on_launch` | integer | Timeout in seconds for the job |
| `job_type` | `ask_job_type_on_launch` | string | Job type (run, check) |

## API Request Examples

### Basic Relaunch (Unchanged)
```json
POST /api/v2/jobs/123/relaunch/
{
  "credential_passwords": {}
}
```

### Enhanced Relaunch with Modified Parameters
```json
POST /api/v2/jobs/123/relaunch/
{
  "credential_passwords": {},
  "inventory": 456,
  "limit": "webservers",
  "extra_vars": {
    "deploy_env": "staging",
    "app_version": "2.1.0"
  },
  "job_type": "check",
  "verbosity": 2,
  "diff_mode": true
}
```

### Error Response for Non-Promptable Fields
```json
HTTP 400 Bad Request
{
  "inventory": ["Field is not allowed to be prompted at launch."],
  "limit": ["Field is not allowed to be prompted at launch."]
}
```

## UI Implementation Requirements

The UI needs to be updated in the ansible-ui repository to:

### 1. Fetch Job Template Configuration
Before showing the relaunch form, fetch the job template to determine which fields are promptable:

```javascript
// Get the job template associated with the job
const jobTemplate = await api.get(`/api/v2/job_templates/${job.job_template}/`);

// Check which fields are promptable
const promptableFields = {
  inventory: jobTemplate.ask_inventory_on_launch,
  limit: jobTemplate.ask_limit_on_launch,
  scm_branch: jobTemplate.ask_scm_branch_on_launch,
  job_tags: jobTemplate.ask_tags_on_launch,
  skip_tags: jobTemplate.ask_skip_tags_on_launch,
  extra_vars: jobTemplate.ask_variables_on_launch,
  verbosity: jobTemplate.ask_verbosity_on_launch,
  diff_mode: jobTemplate.ask_diff_mode_on_launch,
  forks: jobTemplate.ask_forks_on_launch,
  job_slice_count: jobTemplate.ask_job_slice_count_on_launch,
  timeout: jobTemplate.ask_timeout_on_launch,
  job_type: jobTemplate.ask_job_type_on_launch
};
```

### 2. Create Relaunch Form
Create a form that shows only the promptable fields with their current values pre-populated:

```javascript
const RelaunchForm = ({ job, jobTemplate, onSubmit }) => {
  const [formData, setFormData] = useState({
    credential_passwords: {},
    // Pre-populate with current job values
    inventory: job.inventory,
    limit: job.limit,
    extra_vars: job.extra_vars,
    // ... other fields
  });

  const promptableFields = {
    inventory: jobTemplate.ask_inventory_on_launch,
    limit: jobTemplate.ask_limit_on_launch,
    // ... other fields
  };

  return (
    <form>
      {promptableFields.inventory && (
        <InventorySelect
          value={formData.inventory}
          onChange={(value) => setFormData({...formData, inventory: value})}
        />
      )}
      
      {promptableFields.limit && (
        <TextInput
          label="Limit"
          value={formData.limit}
          onChange={(value) => setFormData({...formData, limit: value})}
        />
      )}
      
      {promptableFields.extra_vars && (
        <YamlEditor
          label="Extra Variables"
          value={formData.extra_vars}
          onChange={(value) => setFormData({...formData, extra_vars: value})}
        />
      )}
      
      {promptableFields.job_type && (
        <Select
          label="Job Type"
          value={formData.job_type}
          options={[
            { value: 'run', label: 'Run' },
            { value: 'check', label: 'Check' }
          ]}
          onChange={(value) => setFormData({...formData, job_type: value})}
        />
      )}
      
      {promptableFields.verbosity && (
        <Select
          label="Verbosity"
          value={formData.verbosity}
          options={[
            { value: 0, label: '0 (Normal)' },
            { value: 1, label: '1 (Verbose)' },
            { value: 2, label: '2 (More Verbose)' },
            { value: 3, label: '3 (Debug)' },
            { value: 4, label: '4 (Connection Debug)' },
            { value: 5, label: '5 (WinRM Debug)' }
          ]}
          onChange={(value) => setFormData({...formData, verbosity: value})}
        />
      )}
      
      {promptableFields.diff_mode && (
        <Checkbox
          label="Enable Diff Mode"
          checked={formData.diff_mode}
          onChange={(checked) => setFormData({...formData, diff_mode: checked})}
        />
      )}
      
      {/* Add other promptable fields as needed */}
      
      <Button onClick={() => onSubmit(formData)}>
        Relaunch Job
      </Button>
    </form>
  );
};
```

### 3. Submit Relaunch Request
When the form is submitted, send the request to the relaunch endpoint:

```javascript
const handleRelaunch = async (formData) => {
  try {
    // Only include fields that have been modified or are explicitly set
    const payload = {
      credential_passwords: formData.credential_passwords || {}
    };
    
    // Add only the fields that are different from original or explicitly set
    Object.keys(formData).forEach(key => {
      if (key !== 'credential_passwords' && formData[key] !== undefined) {
        payload[key] = formData[key];
      }
    });
    
    const response = await api.post(`/api/v2/jobs/${job.id}/relaunch/`, payload);
    
    // Navigate to the new job
    navigate(`/jobs/${response.data.id}`);
  } catch (error) {
    // Handle validation errors
    if (error.response?.status === 400) {
      setFormErrors(error.response.data);
    } else {
      setError('Failed to relaunch job');
    }
  }
};
```

### 4. Error Handling
Display validation errors for non-promptable fields:

```javascript
const FormField = ({ field, error, children }) => (
  <div className="form-field">
    {children}
    {error && <div className="field-error">{error}</div>}
  </div>
);

// In the form:
<FormField field="inventory" error={formErrors?.inventory?.[0]}>
  <InventorySelect ... />
</FormField>
```

## Migration Notes

- The existing relaunch functionality remains unchanged for backward compatibility
- Only jobs with promptable fields in their job template will show the enhanced relaunch form
- The API validates all submitted fields against the job template configuration
- Field validation uses the same rules as the original job creation/launch

## Testing

Test cases should cover:
1. Relaunch with no modified fields (existing behavior)
2. Relaunch with modified promptable fields
3. Error handling for non-promptable fields
4. Form field visibility based on job template settings
5. Pre-population of current job values
6. Validation of field types and formats

## Related Files in ansible-ui Repository

The UI changes should be implemented in the ansible-ui repository, likely in files related to:
- Job detail pages
- Job relaunch components
- Form components for job parameters
- API client methods for job operations

This integration guide should help the UI team implement the necessary changes to support the enhanced job relaunch functionality.