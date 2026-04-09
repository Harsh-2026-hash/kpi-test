# kpi-test
KPI management test Repository


week 1,2,3 work done 
currently doing the work of week 4


this repo created by the harsh Rohit just for my project perpose of KPI management
\\make the new commit on the 10-3-2026


 ## Login Feature
  - Added login page
  - Added logout button
  - Fixed session timeout


this is todays committ


from odoo import models, api
import logging
 
_logger = logging.getLogger(__name__)


class GithubCalculator(models.AbstractModel):

    _name        = 'kpi.calculator.github'
    _description = 'GitHub Score Calculator'

    @api.model
    def _calculate_github_score(self,emp,week_start,week_end,weights):
        
        github_records = self.env['kpi.github.activity'].search([
               ('employee_id', '=', emp.id),
               ('date', '>=', week_start),
               ('date', '<=', week_end),
        ])
        if not github_records:
             _logger.warning("%s has no GitHub Data", emp.name)
             return 0.0
        
        total_commits = sum(r.commit_count for r in github_records)
        total_consistency=sum(r.commit_consistency for r in github_records)
        total_pr_opened = sum(r.pr_opened for r in github_records)
        total_pr_merged = sum(r.pr_merged for r in github_records)
        total_lines_added = sum(r.lines_added for r in github_records)
        total_lines_deleted = sum(r.lines_deleted for r in github_records)
        total_issues_closed = sum(r.issues_closed for r in github_records)
        total_reviews = sum(r.reviews_given for r in github_records)


        turnaround_times = [
        r.pr_review_turnaround
        for r in github_records
        if r.pr_review_turnaround > 0
          ]
        avg_turnaround = sum(turnaround_times) / len(turnaround_times) if turnaround_times else 0.0

        calc_days = (week_end - week_start).days + 1 
        t = self.env['kpi.targets']._get_role_targets(emp, days=calc_days)
        
        dynamic_weights = dict(weights)  
        if total_reviews ==0:
            _logger.info("No reviews given by %s. Shifting BOTH review weights to commits.", emp.name)

            w_turnaround = weights.get('avg_review_turnaround', 0)
            w_reviews = weights.get('reviews_given', 0)

            dynamic_weights['avg_review_turnaround'] = 0
            dynamic_weights['reviews_given'] = 0

            dynamic_weights['commit_count'] = dynamic_weights.get('commit_count', 0) + w_turnaround + w_reviews
        
        score = 0.0

        score += self._normalize(total_commits, t.get('commit_count', 0)) * dynamic_weights.get('commit_count', 0) / 100
        score += self._normalize(total_consistency, t.get('commit_consistency', 0)) * dynamic_weights.get('commit_consistency', 0) / 100
        score += self._normalize(total_pr_opened, t.get('pr_opened', 0)) * dynamic_weights.get('pr_opened', 0) / 100
        score += self._normalize(total_pr_merged, t.get('pr_merged', 0)) * dynamic_weights.get('pr_merged', 0) / 100
        score += self._normalize(total_lines_added, t.get('lines_added', 0)) * dynamic_weights.get('lines_added', 0) / 100
        score += self._normalize(total_lines_deleted, t.get('lines_deleted', 0)) * dynamic_weights.get('lines_deleted', 0) / 100
        score += self._normalize(total_issues_closed, t.get('issues_closed', 0)) * dynamic_weights.get('issues_closed', 0) / 100
        score += self._normalize(total_reviews, t.get('reviews_given', 0)) * dynamic_weights.get('reviews_given', 0) / 100
        score += self._normalize(avg_turnaround, t.get('avg_review_turnaround', 0), inverse=True) * dynamic_weights.get('avg_review_turnaround', 0) / 100

        _logger.info("Github Score for %s=%.2f", emp.name, score)
        return round(score, 2)
    
