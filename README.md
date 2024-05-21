# Employee-Performance-Analytics-System


## Employee Performance Analytics System

## Overview:

The HR department is tasked with managing employee performance, optimising salary structures, and reducing employee turnover. However, they face significant challenges in effectively analyzing and utilizing vast amounts of employee-related data to make informed decisions. This Employee Performance Analytics System project aims to address these challenges by providing an SQL-based solution to analyse employee performance, salary distributions, and turnover rates across various departments.

## Business Problem:
The HR department is struggling with the following issues:

1. **Inconsistent Performance Evaluation:** 
   - There is a lack of a unified system to calculate and compare performance scores across departments. This inconsistency makes it difficult to identify high performers and address underperformance.

2. **Unbalanced Salary Structures:**
   - Salary distributions across departments are not well-analyzed, leading to potential inequities and dissatisfaction among employees.

3. **High Employee Turnover:**
   - Certain projects and departments experience higher turnover rates, but the underlying reasons are not clearly understood. This lack of insight hinders the HR department's ability to implement effective retention strategies.

**Project Goals:**
The primary goal of this project is to develop a comprehensive SQL-based analytics system that can:

1. Calculate average performance scores for each department to identify overall departmental performance.
2. Identify the top 5 performers in the company to recognize and reward high achievers.
3. Compare salary distributions across departments to ensure equitable compensation practices.
4. Determine projects with the highest employee turnover rates to understand and address retention issues.

**Key Features:**

- **Tables:** Employees, Departments, Projects, PerformanceReviews, Salaries
- 
- **Queries:**
  - Calculate average performance score for each department.
  - Find the top 5 performers in the company.
  - Compare salary distribution across departments.
  - Determine projects with the highest employee turnover.

By implementing this Employee Performance Analytics System, the HR department will be equipped with the necessary tools and insights to enhance performance evaluation processes, create fair salary structures, and reduce employee turnover, ultimately leading to a more efficient and motivated workforce.

```sql

WITH 
-- CTE to calculate average performance score for each employee along with additional stats
EmployeeAverageScores AS (
    SELECT 
        e.employee_id, 
        e.name, 
        e.department_id, 
        AVG(pr.score) AS average_score,
        COUNT(pr.review_id) AS review_count,
        MIN(pr.score) AS min_score,
        MAX(pr.score) AS max_score,
        STDDEV(pr.score) AS stddev_score
    FROM 
        Employees e
        JOIN PerformanceReviews pr ON e.employee_id = pr.employee_id
    GROUP BY 
        e.employee_id, 
        e.name, 
        e.department_id
),

-- CTE to rank employees within their departments based on their average performance score
EmployeeRankings AS (
    SELECT 
        eas.*, 
        RANK() OVER (PARTITION BY eas.department_id ORDER BY eas.average_score DESC) AS dept_rank,
        DENSE_RANK() OVER (PARTITION BY eas.department_id ORDER BY eas.review_count DESC) AS review_rank
    FROM 
        EmployeeAverageScores eas
),

-- CTE to calculate department-level performance metrics with window functions
DepartmentScores AS (
    SELECT 
        e.department_id, 
        d.department_name, 
        AVG(pr.score) AS average_department_score,
        MIN(pr.score) AS min_department_score,
        MAX(pr.score) AS max_department_score,
        COUNT(DISTINCT e.employee_id) AS num_employees,
        SUM(pr.score) AS total_score,
        STDDEV(pr.score) AS stddev_department_score,
        ROW_NUMBER() OVER (ORDER BY AVG(pr.score) DESC) AS dept_performance_rank
    FROM 
        Employees e
        JOIN PerformanceReviews pr ON e.employee_id = pr.employee_id
        JOIN Departments d ON e.department_id = d.department_id
    GROUP BY 
        e.department_id, 
        d.department_name
),

-- CTE to find the salary distribution quartiles within each department
SalaryDistribution AS (
    SELECT 
        e.department_id, 
        d.department_name, 
        s.salary, 
        NTILE(4) OVER (PARTITION BY e.department_id ORDER BY s.salary) AS salary_quartile,
        PERCENT_RANK() OVER (PARTITION BY e.department_id ORDER BY s.salary) AS salary_percent_rank,
        CUME_DIST() OVER (PARTITION BY e.department_id ORDER BY s.salary) AS salary_cume_dist
    FROM 
        Employees e
        JOIN Salaries s ON e.employee_id = s.employee_id
        JOIN Departments d ON e.department_id = d.department_id
),

-- CTE to calculate project turnover rates with additional statistics
ProjectEmployeeTurnover AS (
    SELECT 
        p.project_id, 
        p.project_name, 
        e.employee_id, 
        COUNT(e.employee_id) OVER (PARTITION BY p.project_id) AS total_employees,
        COUNT(e.termination_date) OVER (PARTITION BY p.project_id) AS turnover_count,
        COUNT(e.employee_id) FILTER (WHERE e.termination_date IS NOT NULL) AS current_turnover,
        AVG(DATE_PART('year', AGE(e.termination_date, e.hire_date))) AS avg_years_before_turnover,
        MAX(e.termination_date) AS latest_turnover_date
    FROM 
        Projects p
        JOIN Employees e ON p.project_id = e.department_id
    WHERE 
        e.termination_date IS NOT NULL
),

-- CTE to rank projects by their turnover rates
ProjectTurnoverRates AS (
    SELECT 
        project_id, 
        project_name, 
        turnover_count, 
        total_employees,
        (turnover_count::float / total_employees) AS turnover_rate,
        RANK() OVER (ORDER BY (turnover_count::float / total_employees) DESC) AS turnover_rank,
        DENSE_RANK() OVER (ORDER BY AVG(DATE_PART('year', AGE(termination_date, hire_date))) DESC) AS avg_tenure_rank
    FROM 
        ProjectEmployeeTurnover
    GROUP BY 
        project_id, 
        project_name, 
        turnover_count, 
        total_employees
)

-- Final SELECT to bring everything together
SELECT 
    ds.department_name, 
    ds.average_department_score, 
    ds.min_department_score,
    ds.max_department_score,
    ds.num_employees,
    ds.total_score,
    ds.stddev_department_score,
    ds.dept_performance_rank,
    er.name AS top_performer_name,
    er.average_score AS top_performer_score,
    er.review_count AS top_performer_review_count,
    er.min_score AS top_performer_min_score,
    er.max_score AS top_performer_max_score,
    er.stddev_score AS top_performer_stddev_score,
    er.dept_rank AS top_performer_dept_rank,
    sd.salary_quartile,
    COUNT(sd.salary) AS salary_count,
    MIN(sd.salary) AS min_salary,
    MAX(sd.salary) AS max_salary,
    AVG(sd.salary) AS avg_salary,
    MAX(sd.salary_percent_rank) AS max_salary_percent_rank,
    MAX(sd.salary_cume_dist) AS max_salary_cume_dist,
    ptr.project_name AS highest_turnover_project,
    ptr.turnover_rate,
    ptr.turnover_rank,
    ptr.avg_tenure_rank,
    ptr.latest_turnover_date
FROM 
    DepartmentScores ds
    JOIN EmployeeRankings er ON ds.department_id = er.department_id AND er.dept_rank = 1
    JOIN SalaryDistribution sd ON ds.department_id = sd.department_id
    JOIN ProjectTurnoverRates ptr ON ds.department_id = ptr.project_id
GROUP BY 
    ds.department_name, 
    ds.average_department_score, 
    ds.min_department_score,
    ds.max_department_score,
    ds.num_employees,
    ds.total_score,
    ds.stddev_department_score,
    ds.dept_performance_rank,
    er.name, 
    er.average_score,
    er.review_count,
    er.min_score,
    er.max_score,
    er.stddev_score,
    er.dept_rank,
    sd.salary_quartile,
    ptr.project_name,
    ptr.turnover_rate,
    ptr.turnover_rank,
    ptr.avg_tenure_rank,
    ptr.latest_turnover_date
ORDER BY 
    ds.average_department_score DESC, 
    ptr.turnover_rate DESC;
